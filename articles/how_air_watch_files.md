---
title: "air"
emoji: "😊"
type: "tech"
topics: []
published: false
---

## airはどのようにファイルを監視するのか

system call使ってるのかなと思い適当に検索してみましたがそれっぽいものは見つからない。

面白そうなのでどのようにファイル監視しているのか見てみる。

```
:) % rg syscall
main.go
9:	"syscall"
62:	signal.Notify(sigs, syscall.SIGINT, syscall.SIGTERM)

runner/engine_test.go
13:	"syscall"
209:			if errors.Is(err, syscall.ECONNREFUSED) {
250:	signal.Notify(sigs, syscall.SIGINT, syscall.SIGTERM)
259:	sigs <- syscall.SIGINT
289:	signal.Notify(sigs, syscall.SIGINT, syscall.SIGTERM)
298:	sigs <- syscall.SIGINT
320:	signal.Notify(sigs, syscall.SIGINT, syscall.SIGTERM)
346:	sigs <- syscall.SIGINT
369:	signal.Notify(sigs, syscall.SIGINT, syscall.SIGTERM)
396:	sigs <- syscall.SIGINT
457:	if errors.Is(err, syscall.ECONNREFUSED) {

runner/util_linux.go
6:	"syscall"
17:		if err = syscall.Kill(-pid, syscall.SIGINT); err != nil {
24:	err = syscall.Kill(-pid, syscall.SIGKILL)

runner/util_darwin.go
7:	"syscall"
16:		if err = syscall.Kill(-pid, syscall.SIGINT); err != nil {
22:	err = syscall.Kill(-pid, syscall.SIGKILL)
31:	c.SysProcAttr = &syscall.SysProcAttr{
```
