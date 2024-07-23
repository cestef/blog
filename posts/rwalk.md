---
title: Scanning the web efficiently
---

## Introduction

In the recent years, I've been taking part in more and more Capture The Flag (CTF) competitions.
One of the most common tasks in these competitions is to find hidden flags on a web server.
This usually involves scanning the web server for hidden files and directories.
This is a very long and boring process, and I felt like the existing tools were not efficient enough.

I decided to write my own tool to scan the web efficiently, and I called it [`rwalk`](https://github.com/cestef/rwalk).

[![rwalk](/images/rwalk.png)](/images/rwalk.png)

I find it very important to have a good understanding of the inner workings of the tools we use, so this article will explain both the process of writing the tool and the features it offers.

## Existing tools

Before starting to write my own tool, let's take a look at the existing ones and see what they offer.

### ffuf

[`ffuf`](https://github.com/ffuf/ffuf) is a fast web fuzzer written in Go.
It is based on a `FUZZ` keyword that is replaced by the words in a wordlist. It can thus be used to scan for parameters, headers, and directories.

```bash copy
ffuf -u http://ffuf.me/cd/basic/FUZZ -w common.txt
```

### Dirsearch

[`dirsearch`](https://github.com/maurosoria/dirsearch) is a simple command line tool designed to brute force directories and files in web servers.

```bash copy
python3 dirsearch.py -u http://ffuf.me/cd/basic/ -w common.txt
```

## Writing the tool

I chose Rust as the programming language for this tool because it's my favorite at the moment and has a great ecosystem for writing command line tools,
with libraries like [`clap`](https://crates.io/crates/clap) and [`reqwest`](https://crates.io/crates/reqwest).

The first functionality I needed was to scan a target recursively. This was achieved by using a simple tree structure.

```rust copy filename="tree.rs"
impl<T> Tree<T> {
    /// Insert a new data into the tree, at the root if no parent provided.
    pub fn insert(
        &mut self,
        data: T,
        parent: Option<Arc<Mutex<TreeNode<T>>>>,
    ) -> Arc<Mutex<TreeNode<T>>> {
        let new_node = Arc::new(Mutex::new(TreeNode {
            data,
            children: Vec::new(),
        }));

        match parent {
            Some(parent) => {
                parent.lock().children.push(new_node.clone());
            }
            None => {
                self.root = Some(new_node.clone());
            }
        }

        new_node
    }

    /// Recursively get all nodes at a given depth
    fn get_nodes_at_depth(
        node: &Option<Arc<Mutex<TreeNode<T>>>>,
        depth: usize,
        nodes: &mut Vec<Arc<Mutex<TreeNode<T>>>>,
    ) {
        if let Some(node) = node {
            if depth == 0 {
                nodes.push(node.clone());
            } else {
                for child in &node.lock().children {
                    Self::get_nodes_at_depth_recursive(&Some(child.clone()), depth - 1, nodes);
                }
            }
        }
    }
}
```

```rust copy filename="single_threaded.rs"
while depth < max_depth {
    let previous_nodes = tree.get_nodes_at_depth(depth);
    // Iterate over the previous nodes
    for previous_node in previous_nodes {
        // Iterate over the wordlist
        for word in wordlist {
            let response = reqwest::blocking::get(&format!("{}/{}", node, word););
            if response.is_ok() {
                let status = response.unwrap().status();
                if status.is_success() {
                    // Add the new node to the tree, with the previous node as its parent
                    tree.insert(word, previous_node);
                }
            }
        }
    }
    // Go to the next depth (/a/b -> /a/b/c)
    depth += 1;
}
```

<figcaption>Pseudocode for the search algorithm</figcaption>

The algorithm is quite simple:

1. We retrieve the nodes at the current depth. (The first time, it's the root node `/`)
2. We iterate over the wordlist and make a request to the server for each word.
3. If the request is successful, we add the new node to the tree, with the previous node as its parent.
4. We go to the next depth.

This single-threaded approach was already quite efficient, but I wanted to make it even faster with multi-threading.
Such an approach requires a bit of refactoring, but Rust makes it easy to do so with the [`tokio`](https://crates.io/crates/tokio) library.

Chunks of the wordlist are distributed to different threads, and the results are collected at the end.

```rust copy filename="multi_threaded.rs" {1,10,18,31-33}
let chunks = wordlist.chunks(wordlist.len() / num_threads);

while depth < max_depth {
    let previous_nodes = tree.get_nodes_at_depth(depth);
    let mut rxs = vec![];
    // Iterate over the previous nodes
    for previous_node in previous_nodes {
        // Iterate over the wordlist
        for chunk in chunks {
            let (tx, rx) = tokio::sync::mpsc::channel(1);

            tokio::spawn(async move {
                for word in chunk {
                    let response = // ...
                    if response.is_ok() {
                        let status = response.unwrap().status();
                        if status.is_success() {
                            tree.insert(word, previous_node);
                        }
                    }
                }
                // Send the signal to the main thread
                tx.send(()).await.unwrap();
            });

            rxs.push(rx);
        }
    }

    // Wait for all the threads to finish
    for mut rx in rxs {
        rx.recv().await;
    }
    // Go to the next depth (/a/b -> /a/b/c)
    depth += 1;
}
```

<figcaption>Pseudocode for the multi-threaded version of the previous algorithm</figcaption>

Notice that the `tx` + `rx` pattern is used to signal the main thread when a thread has finished its work.
We do not need to send any result back as the tree is already wrapped in an `Arc<Mutex<T>>{:rust}` and can be accessed from any thread.

## Adding more features

### Filtering

One of the most important features of a web scanner is to be able to filter the responses.

Basic filters include:

-   Status code
-   Response body
-   Response size
-   Response body hash
-   Response headers
-   Response time

I added those to `rwalk` and made them configurable via the command line.

```rust copy filename="filters.rs" {3,17,19}
let mut outs = vec![];
for filter in filters {
    let negated = filter.0.starts_with('!');
    let out = match filter.0.trim_start_matches('!') {
        "time" => check_range(&parse_range(&filter.1), time) ^ negated,
        "status" => check_range(&parse_range(&filter.1), status_code) ^ negated,
        "contains" => !res_text.contains(&filter.1) ^ negated,
        "starts" => !res_text.starts_with(&filter.1) ^ negated,
        "ends" => !res_text.ends_with(&filter.1) ^ negated,
        "regex" => regex::Regex::new(&filter.1).is_match(res_text) ^ negated,
        "size" => check_range(&parse_range(&filter.1), res_text.len()) ^ negated
        _ => true,
    }
    outs.push(out);
}
if opts.or {
    outs.iter().any(|&x| x)
} else {
    outs.iter().all(|&x| x)
}
```

<figcaption>Pseudocode for the filtering of the responses</figcaption>

The `check_range(){:rust}` and `parse_range(){:rust}` functions are used to parse and check a specific type of filter, which is a range.

| Format       | Description                                               |
| :----------- | :-------------------------------------------------------- |
| `5`          | Exactly `5`                                               |
| `5-10`       | Between `5` and `10` (inclusive)                          |
| `5,10`       | Exactly `5` or `10`                                       |
| `>5`         | Greater than `5`                                          |
| `<5`         | Less than `5`                                             |
| `5,10,15`    | Exactly `5`, `10`, or `15`                                |
| `>5,10,15`   | Greater than `5`, or exactly `10` or `15`                 |
| `5-10,15-20` | Between `5` and `10` or between `15` and `20` (inclusive) |

<figcaption>Range format</figcaption>

This allows for a lot of flexibility when filtering the responses, as most of them require a numeric comparison (more, less, equal, etc.).

For example, let's say we want to filter the responses that only took less than `100` milliseconds:

```ansi
--filter "[0;1mtime:<100[0m"
```

We also needed a way to negate the filters, which is why the `!` character is used to do so with a XOR operation on the base result:

| Filter Output  | Negated        | Result         |
| :------------- | :------------- | :------------- |
| `true{:rust}`  | `false{:rust}` | `true{:rust}`  |
| `false{:rust}` | `false{:rust}` | `false{:rust}` |
| `true{:rust}`  | `true{:rust}`  | `false{:rust}` |
| `false{:rust}` | `true{:rust}`  | `true{:rust}`  |

<figcaption>Truth table for the negation of the filters (XOR)</figcaption>

The default behavior is to `and` all the filters, but the `or` behavior can be enabled with the `--or` flag.
Most of the modern API responses are JSON, and we might want to filter them based on their content.
We need to be able to easily access the JSON fields and compare them to a value.

The following format is used to filter a JSON response:

```ansi
--filter "[0;1mjson:obj.key=value1|value2[0m"
```

To filter only the responses that contain `deadbeef` as the first item of the `data` array:

```ansi
--filter "[0;1mjson:data.0=deadbeef[0m"
```

```json
{
	"data": ["deadbeef", "cafebabe"],
	"other": "stuff",
	"answer": 42
}
```

<figcaption>Example of a JSON response</figcaption>

#### Example

Let's test out these filters with [ffuf.me/cd/no404](http://ffuf.me/cd/no404) !

<small>
	{" "}
	**Note:** See the [install section](#test-it-out-yourself) to install the tool and try it out yourself.{" "}
</small>

We start out with a basic scan as follows:

```bash copy
rwalk http://ffuf.me/cd/no404 common.txt
```

And we get a bunch of **200**s:

```ansi
[0;32m✓[0m [0;1m200[0m http://ffuf.me/cd/no404/wp-dbmanager [0;1;2m142[0;2mms[0m
[0;32m✓[0m [0;1m200[0m http://ffuf.me/cd/no404/header_logo [0;1;2m142[0;2mms[0m
[0;32m✓[0m [0;1m200[0m http://ffuf.me/cd/no404/signon [0;1;2m142[0;2mms[0m
[0;32m✓[0m [0;1m200[0m http://ffuf.me/cd/no404/private2 [0;1;2m142[0;2mms[0m
[0;32m✓[0m [0;1m200[0m http://ffuf.me/cd/no404/mantis [0;1;2m152[0;2mms[0m
[0;32m✓[0m [0;1m200[0m http://ffuf.me/cd/no404/authorization [0;1;2m152[0;2mms[0m
[0;32m✓[0m [0;1m200[0m http://ffuf.me/cd/no404/technical [0;1;2m130[0;2mms[0m
[0;32m✓[0m [0;1m200[0m http://ffuf.me/cd/no404/popup_content [0;1;2m130[0;2mms[0m
[0;32m✓[0m [0;1m200[0m http://ffuf.me/cd/no404/columns [0;1;2m130[0;2mms[0m
[0;32m✓[0m [0;1m200[0m http://ffuf.me/cd/no404/pre [0;1;2m132[0;2mms[0m
[0;32m✓[0m [0;1m200[0m http://ffuf.me/cd/no404/103 [0;1;2m131[0;2mms[0m
[0;32m✓[0m [0;1m200[0m http://ffuf.me/cd/no404/inline [0;1;2m132[0;2mms[0m
[0;32m✓[0m [0;1m200[0m http://ffuf.me/cd/no404/read [0;1;2m142[0;2mms[0m
[0;32m✓[0m [0;1m200[0m http://ffuf.me/cd/no404/ss_vms_admin_sm [0;1;2m142[0;2mms[0m
...
```

Let's make use of the `--show` flag, that allows us to display additional information on each request.

```bash copy
rwalk http://ffuf.me/cd/no404 common.txt --show hash
```

This yields the body hash of each response:

```ansi
[0;32m✓[0m [0;1m200[0m http://ffuf.me/cd/no404/brochures [0;1;2m138[0;2mms[0m | [0;1;2mhash[0m: [0;2m1c3af61e88de6618c0fabbe48edf5ed9[0m
[0;32m✓[0m [0;1m200[0m http://ffuf.me/cd/no404/append [0;1;2m138[0;2mms[0m | [0;1;2mhash[0m: [0;2m1c3af61e88de6618c0fabbe48edf5ed9[0m
[0;32m✓[0m [0;1m200[0m http://ffuf.me/cd/no404/reqs [0;1;2m141[0;2mms[0m | [0;1;2mhash[0m: [0;2m1c3af61e88de6618c0fabbe48edf5ed9[0m
[0;32m✓[0m [0;1m200[0m http://ffuf.me/cd/no404/json [0;1;2m141[0;2mms[0m | [0;1;2mhash[0m: [0;2m1c3af61e88de6618c0fabbe48edf5ed9[0m
[0;32m✓[0m [0;1m200[0m http://ffuf.me/cd/no404/done [0;1;2m146[0;2mms[0m | [0;1;2mhash[0m: [0;2m1c3af61e88de6618c0fabbe48edf5ed9[0m
[0;32m✓[0m [0;1m200[0m http://ffuf.me/cd/no404/command [0;1;2m144[0;2mms[0m | [0;1;2mhash[0m: [0;2m1c3af61e88de6618c0fabbe48edf5ed9[0m
[0;32m✓[0m [0;1m200[0m http://ffuf.me/cd/no404/path [0;1;2m144[0;2mms[0m | [0;1;2mhash[0m: [0;2m1c3af61e88de6618c0fabbe48edf5ed9[0m
[0;32m✓[0m [0;1m200[0m http://ffuf.me/cd/no404/pro [0;1;2m145[0;2mms[0m | [0;1;2mhash[0m: [0;2m1c3af61e88de6618c0fabbe48edf5ed9[0m
[0;32m✓[0m [0;1m200[0m http://ffuf.me/cd/no404/webadm [0;1;2m147[0;2mms[0m | [0;1;2mhash[0m: [0;2m1c3af61e88de6618c0fabbe48edf5ed9[0m
[0;32m✓[0m [0;1m200[0m http://ffuf.me/cd/no404/th [0;1;2m146[0;2mms[0m | [0;1;2mhash[0m: [0;2m1c3af61e88de6618c0fabbe48edf5ed9[0m
[0;32m✓[0m [0;1m200[0m http://ffuf.me/cd/no404/shopstat [0;1;2m146[0;2mms[0m | [0;1;2mhash[0m: [0;2m1c3af61e88de6618c0fabbe48edf5ed9
```

We can now try our luck and search for a response that does not match `1c3af61e88de6618c0fabbe48edf5ed9`:

```bash copy
rwalk http://ffuf.me/cd/no404 common.txt --filter "!hash:1c3af61e88de6618c0fabbe48edf5ed9"
```

And voilà !

```ansi
[0;32m✓[0m [0;1m200[0m http://ffuf.me/cd/no404/secret [0;1;2m114[0;2mms
```

If you are curious, you can also see what was the content of the response with the `--show body` option.

```bash copy
rwalk http://ffuf.me/cd/no404 common.txt --show body --filter "!hash:1c3af61e88de6618c0fabbe48edf5ed9"
```

```ansi
[0;32m✓[0m [0;1m200[0m http://ffuf.me/cd/no404/secret [0;1;2m130[0;2mms[0m | [0;1;2mbody[0m: [0;2mController does not exist
```

### Output

You might want to save the results of your scan to a file, to be able to analyze them with another tool or to share them with someone else.

The available formats are:

-   JSON
-   CSV
-   Markdown
-   Plain text

But as I was using a tree structure to search for the paths, I thought it would be a good idea to print a nice tree a the end of each scan.
The library [`ptree`](https://crates.io/crates/ptree) was perfect for this.

```ansi
[0;32m✓[0m Done in [0;1m7 seconds[0m with an average of [0;1m1990[0m req/s
[0;32m✓[0m [0;2m200[0m /cd/recursion
[0;2m└─ [0;31m✖[0m [0;2m403[0m /admin
[0;2m   └─ [0;31m✖[0m [0;2m403[0m /users
[0;2m      └─ [0;32m✓[0m [0;2m200[0m /96
```

<figcaption>Example of the tree output</figcaption>

### Permutations

In its current state, `rwalk` is only able to scan a target at the end of a path.
But what if we needed to fuzz a path like `/admin/.../login/.../` ?

This is why I added a new scanning mode called `classic`, which allowed for both regular and permutation scanning.
The separation between these modes is due to the fact that they rely on different algorithms.

The permutation scanning mode is based on the [`itertools`](https://crates.io/crates/itertools) crate's `.permutations(){:rust}` method.

```rust copy filename="generate_urls.rs" {2-17}
if opts.permutations {
    let token_count = self
        .url
        .matches(opts.fuzz_key)
        .count();
    let combinations: Vec<_> = words.iter().permutations(token_count).collect();

    combinations
        .iter()
        .map(|c| {
            let mut url = url.clone();
            for word in c {
                url = url.replacen(opts.fuzz_key, word, 1);
            }
            url
        })
        .collect()
} else {
    words
        .iter()
        .map(|w| {
            url.replace(opts.fuzz_key, w)
        })
        .collect()
}
```

<figcaption>URL generation pseudocode</figcaption>

We first compute the number of tokens in the URL, then generate all the `n`-permutations of the wordlist and replace the tokens with the permutations.

### Wordlists

When you need to quickly scan a target, you might not want to use a huge wordlist.

`rwalk` comes with a bunch of filters and transformations to help you generate a wordlist on the fly.

#### Filters

Let's say you need to scan a target for PHP files, you can use the `--filter` flag to only keep the words that end with `.php`.

```ansi
--wordlist-filter [0;1mends=.php[0m
```

or you might want to keep only 4-letter words:

```ansi
--wordlist-filter [0;1msize=4[0m
```

#### Transformations

You might want to transform the words in the wordlist to lowercase, uppercase, or capitalize them.

```ansi
--transform [0;1mlowercase[0m
```

or perhaps you want to replace each instance of `a` with `@`:

```ansi
--transform [0;1mreplace:a=@[0m
```

These two features combined allow for a lot of flexibility when generating a granular wordlist.

You can try this feature with [ffuf.me/cd/ext/logs](http://ffuf.me/cd/ext/logs) and the following command:

```bash copy
rwalk http://ffuf.me/cd/ext/logs common.txt --transform suffix:.log
```

This will generate a wordlist with the `.log` suffix and scan the target for files with this extension.

```ansi
[0;99;31m                    _ _[0m
[0;99;31m                   | | |[0m
[0;99;31m _ ____      ____ _| | | __[0m
[0;99;31m| '__\ \ /\ / / _` | | |/ /[0m
[0;99;31m| |   \ V  V / (_| | |   <[0m
[0;99;31m|_|    \_/\_/ \__,_|_|_|\_\[0m

[0;2mVersion:[0m [0;1;2m0.4.11[0m
[0;2mAuthor:[0m [0;1;2mcstef[0m

[0;34mℹ[0m [0;1m4613[0m words loaded
[0;34mℹ[0m Starting crawler with [0;1m110[0m threads
[0;34mℹ[0m Press [0;1mCtrl+C[0m to save state and exit
[0;32m✓[0m [0;1m200[0m http://ffuf.me/cd/ext/logs/users.log [0;1;2m118[0;2mms[0m
[0;32m✓[0m Done in [0;1m5 seconds[0m with an average of [0;1m1916[0m req/s
[0;31m✖[0m [0;2m403[0m /cd/ext/logs
[0;2m└─ [0;32m✓[0m [0;2m200[0m /users.log
```

### Throttling

When scanning a target, you sometimes need to throttle the requests to avoid being blocked by the server.

Let's take a concrete example with [ffuf.me/cd/rate](http://ffuf.me/cd/rate).

We start with a basic scan:

```bash copy
rwalk http://ffuf.me/cd/rate common.txt
```

This yields us an empty scan result:

```ansi
[0;99;31m                    _ _[0m
[0;99;31m                   | | |[0m
[0;99;31m _ ____      ____ _| | | __[0m
[0;99;31m| '__\ \ /\ / / _` | | |/ /[0m
[0;99;31m| |   \ V  V / (_| | |   <[0m
[0;99;31m|_|    \_/\_/ \__,_|_|_|\_\[0m

[0;2mVersion:[0m [0;1;2m0.4.11[0m
[0;2mAuthor:[0m [0;1;2mcstef[0m

[0;34mℹ[0m [0;1m4613[0m words loaded
[0;34mℹ[0m Starting crawler with [0;1m110[0m threads
[0;34mℹ[0m Press [0;1mCtrl+C[0m to save state and exit
[0;32m✓[0m Done in [0;1m5 seconds[0m with an average of [0;1m2008[0m req/s
[0;32m✓[0m [0;2m200[0m /cd/rate
```

This means that nothing has been matched, let's take a look at the response codes by overriding the default filter:

```bash copy
rwalk http://ffuf.me/cd/rate common.txt --filter status:"100-599"
```

```ansi
[0;31m✖[0m [0;1m429[0m http://ffuf.me/cd/rate/wpau-backup [0;1;2m122[0;2mms[0m
[0;31m✖[0m [0;1m404[0m http://ffuf.me/cd/rate/video [0;1;2m124[0;2mms[0m
[0;31m✖[0m [0;1m429[0m http://ffuf.me/cd/rate/transaction [0;1;2m138[0;2mms[0m
[0;31m✖[0m [0;1m429[0m http://ffuf.me/cd/rate/furl [0;1;2m134[0;2mms[0m
[0;31m✖[0m [0;1m404[0m http://ffuf.me/cd/rate/calling [0;1;2m140[0;2mms[0m
[0;31m✖[0m [0;1m429[0m http://ffuf.me/cd/rate/laptops [0;1;2m136[0;2mms[0m
[0;31m✖[0m [0;1m429[0m http://ffuf.me/cd/rate/resellers [0;1;2m136[0;2mms[0m
[0;31m✖[0m [0;1m429[0m http://ffuf.me/cd/rate/juniper [0;1;2m136[0;2mms[0m
[0;31m✖[0m [0;1m429[0m http://ffuf.me/cd/rate/award [0;1;2m136[0;2mms[0m
[0;31m✖[0m [0;1m404[0m http://ffuf.me/cd/rate/pnadodb [0;1;2m141[0;2mms
```

We can see that we are being rate-limited by the server, so we need to throttle the requests.
The number of requests per second is equal to the throttle times the number of threads.

For example, if we have **110** threads and a throttle of **10**, we will have **1100** requests per second.
Let's go with **10** threads and a **5** second throttle (**50** requests per second).

```bash copy
rwalk http://ffuf.me/cd/rate common.txt --throttle 5 --threads 10
```

After a few minutes, we get the following:

```ansi
[0;99;31m                    _ _[0m
[0;99;31m                   | | |[0m
[0;99;31m _ ____      ____ _| | | __[0m
[0;99;31m| '__\ \ /\ / / _` | | |/ /[0m
[0;99;31m| |   \ V  V / (_| | |   <[0m
[0;99;31m|_|    \_/\_/ \__,_|_|_|\_\[0m

[0;2mVersion:[0m [0;1;2m0.4.11[0m
[0;2mAuthor:[0m [0;1;2mcstef[0m

[0;34mℹ[0m [0;1m4613[0m words loaded
[0;34mℹ[0m Starting crawler with [0;1m10[0m threads
[0;34mℹ[0m Press [0;1mCtrl+C[0m to save state and exit
[0;32m✓[0m [0;1m200[0m http://ffuf.me/cd/rate/oracle [0;1;2m202[0;2mms[0m
[0;32m✓[0m Done in [0;1m2 minutes[0m with an average of [0;1m49[0m req/s
[0;32m✓[0m [0;2m200[0m /cd/rate
[0;2m└─ [0;32m✓[0m [0;2m200[0m /oracle
```

## Final result

We now have a tool that is able to scan a target efficiently, approaching **2000** requests per second on a regular WiFi connection.

[![rwalk](/images/rwalk.gif)](/images/rwalk.gif)

### Test it out yourself

You can find the source code on [GitHub](https://github.com/cestef/rwalk).

The tool can be installed with [`brew`](https://brew.sh) or [`cargo`](https://rustup.rs):

```bash copy
brew install cestef/tap/rwalk
```

or

```bash copy
cargo install rwalk
```

I like to use [ffuf.me](http://ffuf.me) to test out the tool, as it's a great target for web scanning.

A good wordlist to start with is [common.txt](https://github.com/danielmiessler/SecLists/blob/master/Discovery/Web-Content/common.txt).

```bash copy
rwalk http://ffuf.me/cd/recursion common.txt --depth 3
```

<figcaption className="mb-6">Recursive scan of ffuf.me</figcaption>

```bash copy
rwalk http://ffuf.me/cd/basic/$ common.txt --mode classic
```

<figcaption>Classic scan of ffuf.me</figcaption>

I hope you find this tool as useful as I do 😄

Feel free to open an [issue](https://github.com/cestef/rwalk/issues) or join the [Discord](https://cstef.dev/discord) if you have any questions or suggestions !
