# Go By Doing

## Introduction
I have not used go extensively. That said, it is very fast and offers a lot of really cool
features. Easy concurrency, nice and fast, a solid compiler, robust serverside libraries.
It seems as if it would be silly not to learn it. Yesterday, while perusing [Facebook's](http://www.github.com/facebook)
React examples, I encountered a simple server written in Go.

```GO
package main
import (
"bytes"
"encoding/json"
"fmt"
"io"
"io/ioutil"
"log"
"net/http"
"os"
"sync"
)
type comment struct {
Author string `json:"author"`
Text string `json:"text"`
}
const dataFile = "./comments.json"
var commentMutex = new(sync.Mutex)
// Handle comments
func handleComments(w http.ResponseWriter, r *http.Request) {
// Since multiple requests could come in at once, ensure we have a lock
// around all file operations
commentMutex.Lock()
defer commentMutex.Unlock()
// Stat the file, so we can find it's current permissions
fi, err := os.Stat(dataFile)
if err != nil {
http.Error(w, fmt.Sprintf("Unable to stat the data file (%s): %s", dataFile, err), http.StatusInternalServerError)
return
}
// Read the comments from the file.
commentData, err := ioutil.ReadFile(dataFile)
if err != nil {
http.Error(w, fmt.Sprintf("Unable to read the data file (%s): %s", dataFile, err), http.StatusInternalServerError)
return
}
switch r.Method {
case "POST":
// Decode the JSON data
comments := make([]comment, 0)
if err := json.Unmarshal(commentData, &comments); err != nil {
http.Error(w, fmt.Sprintf("Unable to Unmarshal comments from data file (%s): %s", dataFile, err), http.StatusInternalServerError)
return
}
// Add a new comment to the in memory slice of comments
comments = append(comments, comment{Author: r.FormValue("author"), Text: r.FormValue("text")})
// Marshal the comments to indented json.
commentData, err = json.MarshalIndent(comments, "", " ")
if err != nil {
http.Error(w, fmt.Sprintf("Unable to marshal comments to json: %s", err), http.StatusInternalServerError)
return
}
// Write out the comments to the file, preserving permissions
err := ioutil.WriteFile(dataFile, commentData, fi.Mode())
if err != nil {
http.Error(w, fmt.Sprintf("Unable to write comments to data file (%s): %s", dataFile, err), http.StatusInternalServerError)
return
}
w.Header().Set("Content-Type", "application/json")
w.Header().Set("Cache-Control", "no-cache")
io.Copy(w, bytes.NewReader(commentData))
case "GET":
w.Header().Set("Content-Type", "application/json")
w.Header().Set("Cache-Control", "no-cache")
// stream the contents of the file to the response
io.Copy(w, bytes.NewReader(commentData))
default:
// Don't know the method, so error
http.Error(w, fmt.Sprintf("Unsupported method: %s", r.Method), http.StatusMethodNotAllowed)
}
}
func main() {
http.HandleFunc("/comments.json", handleComments)
http.Handle("/", http.FileServer(http.Dir("./public")))
log.Println("Server started: http://localhost:3000")
log.Fatal(http.ListenAndServe(":3000", nil))
}

```

[source](https://github.com/reactjs/react-tutorial/blob/master/server.go)

And also provided was a Python equivalent

```PYTHON
...
import json
from flask import Flask, Response, request
app = Flask(__name__, static_url_path='', static_folder='public')
app.add_url_rule('/', 'root', lambda: app.send_static_file('index.html'))
@app.route('/comments.json', methods=['GET', 'POST'])
def comments_handler():
with open('comments.json', 'r') as file:
comments = json.loads(file.read())
if request.method == 'POST':
comments.append(request.form.to_dict())
with open('comments.json', 'w') as file:
file.write(json.dumps(comments, indent=4, separators=(',', ': ')))
return Response(json.dumps(comments), mimetype='application/json', headers={'Cache-Control': 'no-cache'})
if __name__ == '__main__':
app.run(port=3000)
```
[source](https://github.com/reactjs/react-tutorial/blob/master/server.py)

90 some odd lines, and very veebose, vs a grand total of just over 30? This doesn't really make a good
argument for it on any clear level outside of the obvious concurrency we are getting via the comment
mutex's, etc. However, writing a server in a compiled language really *should* just take more code,
theoretically. 

Go has some of the most eloquently written source code I have ever seen. This, while functional and
working is not nearly as well written as pieces of Go I have read from it's own source. Knowing this,
I thought another approach of understanding Go may be to dive into it's source of a module similar
to Python's. In this case, I chose [requests](http://docs.python-requests.org/en/latest/), for it is
the most similar thing to Go's Net/HTTP standard library pieces.
