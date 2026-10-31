# bloppadd

***bloppadd*** is the 25kB KISS client to llama.cpp's **llama-server**.

![image](https://github.com/clort81/bloppadd/blob/main/bloppadd.gif)

Type things, hit Ctrl-D to send, get responses back.  Paste to input box, copy from output box.

**NEW!** Undo+Edit last, Save Last, Image Upload

## Why this thing? 

Llama.cpp WebUI slows down to 0.4 characters per second when the page gets large. Alternative frontends like [mikupad](https://github.com/lmg-anon/mikupad) existings, but they built with ReactJS and needs a compile step and is like 400kB.

Is needing`./compile.sh` to get webpage cool?  *I don't think so.*

Enter **bloppadd!** because:
- browsers already have a text editor (it's called `<textarea>`)
- browsers already know how to make HTTP requests (it's called `fetch()`)

bloppadd is like 25kB.  It doesn't track your keystrokes and slow down to 0.4 chars per second like llama.cpp WebUI. It doesn't need `npm install`.  No Node.js. No transpilation. No bundling. No virtual DOM. No dependencies to audit for security vulnerabilities every tuesday.

**"But React is the future!" Yeah great, okay ... okay.. bye.**

## usage

1. start llama-server
2. open bloppadd.html in firefox or chrome
3. type stuff
4. Ctrl+D to generate

Problems? The italians decided that since stupid users open malicious locally stored .html programs they should block those files from working.  
Andrea Marchesini (baku) and landed in Firefox 68 (July 2019) in response to CVE-2019-11730, reported by security researcher Luigi Gubello.
If you've reviewed this and decided it's not a malicious program, then in Firefox about:config → security.fileuri.strict_origin_policy → false

## features

- single file
- no build step
- actually lightweight
- respects your keyboard
- shows tokens/sec stats
- works offline forever
- will still run in 20 years when React is archaeological dust

## comparing to mikupad

| thing | mikupad | bloppadd |
|-------|---------|----------|
| size | 400kB+ | 24.6kB |
| build required | yes (compile.sh) [[1]] | no |
| framework | ReactJS [[1]] | none |
| dependencies | React, build tools | browser |
| loads fast | no | yes |
| will survive npm apocalypse | no | yes |

## Contributing

NO

## License

MIT Public License.  Made with LOCAL LLM ONLY: Qwen27b and GLM5.3-flash.

## the end

I'm the best!
(made humbly with local llama: qwen27b and glm5.3-flash)              
