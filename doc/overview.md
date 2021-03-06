# Writing a Chatbot with Clojure

* Ray Miller
* Jim Downing

---

## Overview

1. Getting started at the REPL
2. Clojure as a REST client
  - Interacting with the TheySaidSo API
3. Posting messages to Slack
  - Simple messages
  - Message formatting and attachments
4.  A simple Clojure web server
5. Handling Slack commands
6. Putting the pieces together
  - Deploying to Heroku
  - Plumbing the bits together

---

## Getting started at the REPL

Install Leiningen: http://leiningen.org/#install

```bash
lein new hello-world
cd hello-world
lein repl
```

```clojure-repl
user=> (println "Hello World")
```

---
## Clojure as a REST client

---
### Clojure as a REST client 1/5: Dependencies

Edit `project.clj` and add the following to dependencies:

```clojure
[clj-http "3.1.0"]
[cheshire "5.6.1"]
```

Restart the REPL:

```bash
lein repl
```

---
### Clojure as a REST client 2/5: Our first request

```clojure
(require '[clj-http.client :as http] '[clojure.pprint :refer [pprint]])
(def response (http/get "http://quotes.rest/qod.json"))
(pprint response)
```

---

### Clojure as a REST client 3/5: Parsing the body

```clojure
(def response (http/get "http://quotes.rest/qod.json" {:as :json}))
(pprint response)
(get-in response [:body :contents :quotes 0 :quote])
```

---
### Clojure as a REST client 4/5: Query parameters

```clojure
(def categories (http/get "http://quotes.rest/qod/categories.json" {:as :json}))
(get-in categories [:body :contents :categories])
(pprint (http/get "http://quotes.rest/qod.json"
                  {:as :json :query-params {:category "art"}}))
```

---
### Clojure as a REST client 5/5: Setting request headers

```clojure
(def api-key "h7GmOyL2jnVBjDgNyoaEDAeF")
(http/get "http://quotes.rest/qod.json"
          {:as :json
           :query-params {:category "life"}
           :headers {"X-Theysaidso-API-Secret" api-key}})
```

---
## Posting messages to Slack 1/2

Configure an incoming WebHook:

https://dev-summer-cb.slack.com/apps/manage/

Make a note of the WebHook URL:

```clojure-repl
user=> (def webhook-url "https://hooks.slack.com/services/...")
```

Post a message to Slack:

```clojure-repl
user=> (http/post webhook-url {:form-params {:text "My first message"}
                                :content-type :json})
```

---
### Posting messages to Slack 2/2: Advanced message formatting

https://api.slack.com/docs/message-attachments

```clojure-repl
user=> (def params
         {:attachments
          [{:fallback
            "New ticket from Andrea Lee - Ticket #1943: Can't rest my password - https://groove.hq/path/to/ticket/1943",
            :pretext "New ticket from Andrea Lee",
            :title "Ticket #1943: Can't reset my password",
            :title_link "https://groove.hq/path/to/ticket/1943",
            :text "Help! I tried to reset my password but nothing happened!",
            :color "#7CD197"}]})
user=> (http/post webhook-url {:form-params params :content-type :json})
```

---
## Implementing a web service in Clojure

---
### Web Server 1/N: Ring

* Clojure’s equivalent of Ruby’s Rack, Python’s WSGI and Perl’s Plack
* Deals with the Java Servlet API
* Provides utilities for parsing requests and generating responses
* Extensible through middleware

---
### Web Server 2/N: Our first Ring application

```bash
lein new compojure chatbot
cd chatbot
lein ring server
```

---
### Web Server 3/N: Our first Ring application

```clojure
(ns chatbot.handler
  (:require [compojure.core :refer :all]
            [compojure.route :as route]
            [ring.middleware.defaults :refer [wrap-defaults site-defaults]]))

(defroutes app-routes
  (GET "/" [] "Hello World")
  (route/not-found "Not Found"))

(def app
  (wrap-defaults app-routes site-defaults))
```

---
### Web Server 4/N: The request map

Update `project.clj`:

```clojure
{:dev {:dependencies [[javax.servlet/servlet-api "2.5"]
                      [ring/ring-mock “0.3.0”]
*                      [ring/ring-devel “1.4.0”]]}}
```

Edit `chatbot/handler.clj`:

```clojure
(ns chatbot.handler
  (:require [compojure.core :refer :all]
            [compojure.route :as route]
            [ring.middleware.defaults
                :refer [wrap-defaults site-defaults]]
*            [ring.handler.dump :refer [handle-dump]]))

(defroutes app-routes
  (GET "/" [] "Hello World")
*  (GET "/dump/*" [] handle-dump)
  (route/not-found "Not Found"))

(def app
  (wrap-defaults app-routes site-defaults))
```

* Restart your app and make some requsets to <http://localhost:3000/dump/>
* Try adding query parameters to the URL

---
### Web Server 5/N: The response map

```clojure
(defn my-handler
  [request]
  {:body "Hello World"
   :status 200
   :headers {"Content-Type" "text/plain"}})

(defroutes app-routes
  (GET "/" [] my-handler)
  (route/not-found "Not Found"))
```

---
### Web Server 6/N: response utility

```clojure-repl
user=> (require '[ring.util.response :refer [response status charset content-type]])
user=> (response "Hello World")
user=> (-> (responsne "Not found") (status 404) (content-type "text/plain"))
```

---
### Web Server 7/N: Middleware

A ring handler is simply a function that receives a request map and returns a response map.

Ring middleware is simply a function that modifies the behaviour of a
handler. It can run before the handler and modify the request map, or
after the handler and modify the response map.

---
### Web Server 8/N: Request Middleware

```clojure
(defn wrap-check-token
  [handler]
  (fn [request]
    (if (= (get-in request [:params :token]) slack-token)
      (handler request)
      (-> (response "Invalid token")
          (status 400)))))
```

---
### Web Server 9/N: Response Middleware

```clojure
(defn wrap-json-response
  [handler]
  (fn [request]
    (let [res (handler request)]
      (if (coll? (:body res))
        (-> res
            (update :body json/generate-string)
            (content-type "application/json")
            (charset "UTF-8"))
        res))))
		```

---
## Handling slack commands

https://api.slack.com/slash-commands

---
## Deploying to Heroku

- Install Heroku Toolbelt
- Login and create app
- Configure leiningen integration
- Deploy and start app
- Testing

---

### Heroku 1/5: Install Heroku Toolbelt

https://toolbelt.heroku.com/

---

### Heroku 2/5: Login and create app
```bash
$ heroku login
Enter your Heroku credentials.
Email: adam@example.com
Password (typing will be hidden):
Authentication successful.

$ cd ~/myapp
$ heroku create
Creating vast-citadel-38177... done, stack is cedar-14
http://vast-citadel-38177.herokuapp.com/ | https://git.heroku.com/vast-citadel-38177.git
Git remote heroku added
```

---

### Heroku 3/5: Configure leiningen integration

Most of this has already been done for you in project.clj

```clojure
:plugins [[lein-ring "0.9.7"]
          [lein-heroku "0.5.3"]]
* :heroku {:app-name "young-thicket-18780"
         :jdk-version "1.8"
         :include-files ["target/chatbot-0.1.0-SNAPSHOT-standalone.jar"]
         :process-types {"web" "java -jar target/chatbot-0.1.0-SNAPSHOT-standalone.jar"}}
```

You just need to customize the app-name on the highlighted line.

???
TODO: Would be good to get this out of project.clj

---

### Deploy and start app
Deploy:
```bash
lein ring uberjar
lein heroku deploy-uberjar
```
Start:
```bash
heroku ps:scale web=1 -a vast-citadel-38177
```

---

### Testing

```clojure
(require '[clj-http.client :as http] '[chatbot.handler])

(http/post "https://vast-citadel-38177.herokuapp.com/slack"
           {:form-params {:command "/quote"
                          :token chatbot.handler/slack-token}})
```

---

## Putting the pieces together

Create Slack command integration, use URL for your Heroku app:

https://vast-citadel-38177.herokuapp.com/slack
