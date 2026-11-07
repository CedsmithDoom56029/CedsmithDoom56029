# pb

**A localhost pastebin in one Go file. Zero dependencies, stdlib only.**
Pipe text in, get a URL back. Self-destructs after 24 h.

```
$ echo "deploy token: s3cr3t" | curl --data-binary @- localhost:8080
http://localhost:8080/x2fK9aB
```

## Why?

- You need to move a log, a snippet or a config between two terminals / two machines on LAN.
- `nc` is awkward, chat apps leak, and installing a real pastebin needs a database and a philosophy degree.
- `pb` is ~70 lines of stdlib Go. Paste it anywhere Go compiles.

## Quick start

1. Copy the code below into `pb.go`.
2. `go build pb.go` (or just `go run pb.go`).
3. Paste things:

```bash
# pipe anything
cat /var/log/app.log | curl --data-binary @- localhost:8080

# get it back
curl localhost:8080/x2fK9aB

# save straight to a file
curl localhost:8080/x2fK9aB -o app.log
```

## The whole program

```go
package main

import (
	"crypto/rand"
	"encoding/base64"
	"fmt"
	"io"
	"net/http"
	"os"
	"path/filepath"
	"strings"
	"time"
)

const (
	addr = ":8080"
	dir  = ".pb"
	ttl  = 24 * time.Hour
	max  = 2 << 20 // 2 MiB per paste
)

func key() string {
	b := make([]byte, 6)
	rand.Read(b)
	return base64.RawURLEncoding.EncodeToString(b)
}

func main() {
	os.MkdirAll(dir, 0o755)
	go janitor()

	http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
		id := strings.TrimPrefix(r.URL.Path, "/")

		if r.Method == http.MethodPost && id == "" {
			r.Body = http.MaxBytesReader(w, r.Body, max)
			data, err := io.ReadAll(r.Body)
			if err != nil {
				http.Error(w, "too big", http.StatusRequestEntityTooLarge)
				return
			}
			if len(data) == 0 {
				http.Error(w, "empty paste", http.StatusBadRequest)
				return
			}
			id = key()
			os.WriteFile(filepath.Join(dir, id), data, 0o600)
			fmt.Fprintf(w, "http://localhost%s/%s\n", addr, id)
			return
		}

		if id == "" || strings.ContainsAny(id, `/\`) {
			http.Error(w, "POST something first", http.StatusNotFound)
			return
		}
		data, err := os.ReadFile(filepath.Join(dir, id))
		if err != nil {
			http.NotFound(w, r)
			return
		}
		if strings.HasPrefix(http.DetectContentType(data), "text/") {
			w.Header().Set("Content-Type", "text/plain; charset=utf-8")
		} else {
			w.Header().Set("X-Content-Type-Options", "nosniff")
		}
		w.Write(data)
	})

	fmt.Println("pb listening on http://localhost" + addr)
	http.ListenAndServe(addr, nil)
}

func janitor() {
	for range time.Tick(time.Hour) {
		entries, _ := os.ReadDir(dir)
		cutoff := time.Now().Add(-ttl)
		for _, e := range entries {
			if info, err := e.Info(); err == nil && info.ModTime().Before(cutoff) {
				os.Remove(filepath.Join(dir, e.Name()))
			}
		}
	}
}
```

## How it works

- `POST /` — read the body (max 2 MiB), generate a random 8-char id, store it in `.pb/`, return the URL.
- `GET /<id>` — serve it back with the content type sniffed by `http.DetectContentType`.
- A janitor goroutine wakes up hourly and deletes pastes older than 24 h.
- Random ids come from `crypto/rand`, not a counter — so nobody can enumerate your pastes.

## Caveats

- Binds to all interfaces on `:8080`. It's for localhost/LAN dev, not for the internet.
- The returned URL says `localhost` — replace the host with your LAN IP to share across machines.
- No auth, no encryption, no guarantees. It's a clipboard with a URL, not a vault.

## License

MIT. If you fork it to add a web UI, you have been warned.
