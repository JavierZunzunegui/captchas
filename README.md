# Captchas
[![Build Status](https://travis-ci.org/clevergo/captchas.svg?branch=master)](https://travis-ci.org/clevergo/captchas)
[![Coverage Status](https://coveralls.io/repos/github/clevergo/captchas/badge.svg?branch=master)](https://coveralls.io/github/clevergo/captchas?branch=master)
[![GoDoc](https://img.shields.io/badge/godoc-reference-blue)](https://pkg.go.dev/github.com/clevergo/captchas)
[![Go Report Card](https://goreportcard.com/badge/github.com/clevergo/captchas)](https://goreportcard.com/report/github.com/clevergo/captchas)
[![Release](https://img.shields.io/github/release/clevergo/captchas.svg?style=flat-square)](https://github.com/clevergo/captchas/releases)

Base64 Captchas Manager, supports multiple [drivers](#drivers)(digit, math, audio, string, chinese etc.) and [stores](#stores)(memory, redis, memcached etc.).

## Usage

Preview: http://129.204.189.159:10000/.

```shell
$ cd example
$ go run main.go
```

### Quick Start

```go
package main

import (
	"encoding/json"
	"fmt"
	"io"
	"log"
	"net/http"
	"text/template"

	"github.com/clevergo/captchas"
	"github.com/clevergo/captchas/drivers"
	"github.com/clevergo/captchas/memstore"
)

var (
	store   = memstore.New()              // memory store.
	driver  = drivers.NewDigit()          // digit driver.
	manager = captchas.New(store, driver) // manager
	tmpl    = template.Must(template.New("captcha").Parse(`
<html>
<body>
<form action="/validate" method="POST">
	<input name="captcha">
	{{ .captcha.HTMLField "captcha_id" }}
	<input type="submit" value="Submit">
</form>
</body>
</html>
	`))
)

func main() {
	http.HandleFunc("/generate", generate)
	http.HandleFunc("/validate", validate)
	log.Println(http.ListenAndServe(":8080", http.DefaultServeMux))
}

// generates a new captcha
func generate(w http.ResponseWriter, r *http.Request) {
	captcha, err := manager.Generate()
	if err != nil {
		http.Error(w, err.Error(), 500)
	}

	// returns JSON data.
	if r.URL.Query().Get("format") == "json" {
		v := map[string]string{
			"captcha_id":   captcha.ID(),             // captcha ID.
			"captcha_data": captcha.EncodeToString(), // base64 encode string.
		}
		data, _ := json.Marshal(v)
		w.Write(data)
		return
	}

	// render captcha via template.
	tmpl.Execute(w, map[string]interface{}{
		"captcha": captcha,
	})

}

// validates a captcha.
func validate(w http.ResponseWriter, r *http.Request) {
	captchaID := r.PostFormValue("captcha_id")
	captcha := r.PostFormValue("captcha")

	// verify
	if err := manager.Verify(captchaID, captcha, true); err != nil {
		io.WriteString(w, fmt.Sprintf("captcha is invalid: %s", err.Error()))
		return
	}

	io.WriteString(w, "valid")
}
```

## Drivers

```go
import "github.com/clevergo/captchas/drivers"
```

### Digit

```go
// all options are optional.
opts := []drivers.DigitOption{
	drivers.DigitHeight(50),
	drivers.DigitWidth(120),
	drivers.DigitLength(6),
	drivers.DigitMaxSkew(0.8),
	drivers.DigitDotCount(80),
}
driver := drivers.NewDigit(opts...)
```

### Audio

```go
// all options are optional.
opts := []drivers.AudioOption{
	drivers.AudioLangauge("en"),
	drivers.AudioLength(6),
}
driver := drivers.NewAudio(opts...)
```

### Math

```go
// all options are optional.
opts := []drivers.MathOption{
	drivers.MathHeight(50),
	drivers.MathWidth(120),
	drivers.MathNoiseCount(0),
	drivers.MathFonts([]string{}),
	drivers.MathBGColor(&color.RGBA{}),
}
driver := drivers.NewMath(opts...)
```

### String

```go
// all options are optional.
opts := []drivers.StringOption{
	drivers.StringHeight(50),
	drivers.StringWidth(120),
	drivers.StringLength(4),
	drivers.StringNoiseCount(0),
	drivers.StringFonts([]string{}),
	drivers.StringSource("abcdefghijklmnopqrstuvwxyz"),
	drivers.StringBGColor(&color.RGBA{}),
}
driver := drivers.NewString(opts...)
```

### Chinese

```go
// all options are optional.
opts := []drivers.ChineseOption{
	drivers.ChineseHeight(50),
	drivers.ChineseWidth(120),
	drivers.ChineseLength(4),
	drivers.ChineseNoiseCount(0),
	drivers.ChineseFonts([]string{"wqy-microhei.ttc"}),
	drivers.ChineseSource("零一二三四五六七八九十"),
	drivers.ChineseBGColor(&color.RGBA{}),
}
driver := drivers.NewChinese(opts...)
```

## Stores

- [memory](#memory)
- [redis](#redis)
- [memcached](#memcached)
- add your store here by PR or [request a new store](https://github.com/clevergo/captchas/issues/new).

### Memory

```go
import "github.com/clevergo/captchas/memstore"
```

```go
store := memstore.New(
	memstore.Expiration(10*time.Minute), // captcha expiration, optional.
	memstore.GCInterval(time.Minute), // garbage collection interval to delete expired captcha, optional.
)
```

> Inspired by [scs.memstore](https://github.com/alexedwards/scs/tree/master/memstore).

### Redis

```go
import (
    "github.com/clevergo/captchas/redisstore"
    "github.com/go-redis/redis/v7"
)
```

```go
// redis client.
client := redis.NewClient(&redis.Options{
	Addr: "localhost:6379",
})
store := redisstore.New(
	client,
	redisstore.Expiration(expiration), // captcha expiration, optional.
	redisstore.Prefix("caotchas"), // redis key prefix, optional.
)
```

### Memcached

```go
import (
	"github.com/bradfitz/gomemcache/memcache"
	"github.com/clevergo/captchas/memcachedstore"
)
```

```go
// client.
client := memcache.New("localhost:11211")
store := memcachedstore.New(
	client,
	memcachedstore.Expiration(int32(600)), // captcha expiration, optional.
	memcachedstore.Prefix("captchas"),     // key prefix, optional.
)
```

