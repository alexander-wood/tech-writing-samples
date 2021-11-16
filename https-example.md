# Creating A Webserver in Node and Express
## Introduction

This guide will walk you through configuring a webserver using Node.js. We will install Node.js from scratch, use a package called Express to serve a JSON file, and configure HTTPS on a server using a self-signed certificate.

Node.js is a Javascript-based environment which you can use to create webservers and networked applications. It comes with NPM, the Node Package Manager, which makes installing Node "packages" fast and easy. 

Express is a Node package which is a fast, unopinionated, minimalist web framework.
 
## Prerequisites
This guide assumes that you're using an **OSX** environment with **Homebrew**, **curl**, and **openssl** installed. It also assumes that you're comfortable working on the command line.

If you need help installing Homebrew, please refer to the official documentation [here](https://brew.sh/).

On OSX, you can install curl and openssl by using Homebrew: `brew install curl` and `brew install openssl`.

## Installing Node.js
Installing Node.js and NPM is straightforward using Homebrew. Homebrew handles downloading, unpacking, and installing Node and NPM on your system. To install:
* **Open your terminal** and run `brew update` to make sure Homebrew has the latest versions of Node available.
* **Install Node** by running `brew install node`. This may take a few minutes.

### Test it!
To make sure that Homebrew installed Node correctly, run the following commands to check Node and NPM's version.
* **Verify Node** by running `node -v`. You should see something like `v17.0.1`.
* **Verify NPM** by running `npm -v`. You should see something like `8.1.0`.
   
## Installing Express.js
Once Node.js and NPM are installed, installing Express.js is a straightforward process.
* **Create a directory** to house the application and then navigate to it by running `mkdir express-example && cd express-example`. 
* **Create a new Node.js application** by running `npm init`. This command will prompt you for a project name, version number, the name of the main code file for the project, and a few other details. For now, it's acceptable to use the default values for everything. 
 
  `npm init` will create a `package.json` file in the root of your project directory. `package.json` specifies important information, including dependencies. 

* **Install Express** by running `npm install express --save`. This will add it to the dependencies list in `package.json`.

Now that you've got Node and Express installed, it's time to create the webserver!

## Creating A Webserver
### Creating the `example.json` file
JSON stands for Javascript Object Notation. It is a common format for transferring data on the web. Objects in Javascript are represented as a collection of keys and values. Let's create a simple `.json` file for our webserver to serve as a response.

* **Create the JSON file** with `touch example.json`.
* **Open** `example.json` in the editor of your choice.
* **Add the following code** to `example.json`:
  ```json
  {
      "message": "Hello world!"
  }
  ```

Now that there's some data to serve, let's write the code for our webserver!

### Creating the `index.js` file
 When you initialized the project with `npm init`, it asked you to name a file as the main file of the app. By default, this file is named `index.js`. This will be the file where our application code lives.

* **Create the file for the server** with `touch index.js`. 
* **Open** `index.js` in the editor of your choice.
* **Add the following code** to `index.js`:
  ```js
    const express = require('express')
    const app = express()
    const port = 3000

    app.get('/', (req, res) => {
      const data = require('./example.json')
      res.json(data);
    })

    app.listen(port, () => {
      console.log(`Example app listening at http://localhost:${port}`);
    })
  ```
This is all the code needed to set up a webserver in Express! 

The first three lines tell the program to use Express and instantiate it under the name `app`. 

The second block of code tells `app` to handle incoming requests to the root route by responding with the contents of `example.json`. 

The final block of code tells `app` what port to run on and provides a user-friendly startup message.

Now we can start up our server and make our first requests to it!

### Seeing the Server In Action
  * **Start the server** by running `node index.js`. You should see `Example app listening at http://localhost:3000`.
  * **Send an HTTP request** by running `curl http://localhost:3000`. You should get the response `{ "message": "Hello world!" }`.
  * **Send an HTTPS request** by running `curl https://localhost:3000`. You should get an error like `curl: (35) error:1400410B:SSL routines:CONNECT_CR_SRVR_HELLO:wrong version number`. We have yet to configure HTTPS, so this is normal!
  * **When you're finished with the server**, close it by pressing `control+c`.

Now that we have a functioning web server which can serve JSON from a file, we're ready to update our server to use HTTPS.

## Configuring HTTPS
### Introduction to HTTPS
HTTPS stands for Hypertext Transfer Protocol Secure. It is a way of encrypting the HTTP traffic to protect the privacy and security of the exchanged data.

Using HTTPS in a production system requires a trusted third-party to sign digital certificates verifying that the server is trusted. These third-parties are called Certificate Authorities. Certificate Authorities are often commercial enterprises offering paid-for TLS certificates. 

For development purposes, it is acceptable to use a self-signed TLS certificate.

### Installing Self-Signed HTTPS Certificate
To generate a self-signed TLS certificate, we'll use a program called `openssl` which lets anyone create their own TLS certificate. This is a two-step process:
* **Generate A Public Key Certificate** by running `openssl req -x509 -newkey rsa:2048 -keyout keytmp.pem -out cert.pem -days 365`. You'll be prompted to enter a passphrase and some location data. Use a passphrase that you'll remember. It's acceptable to use the default value for everything else.
* **Generate Public Keys** by running `openssl rsa -in keytmp.pem -out key.pem`. This will ask you to enter the passphrase you created in the previous command.
* **Verify the keys have been created** by running `ls` in your project directory. You should see `key.pem` and `cert.pem` in the directory.
  
Once you've got a self-signed certificate and public keys, you're ready to adjust the webserver's code to handle HTTPS!

### Updating the Webserver Code
Changing the webserver code to handle HTTPS is straightforward. Open `index.js` and update the code to the following:

```js
  const express = require('express')
  const fs = require('fs')
  const https = require('https')
  const key = fs.readFileSync('./key.pem')
  const cert = fs.readFileSync('./cert.pem')
  const app = express()
  const port = 3000
  const server = https.createServer({key: key, cert: cert }, app);

  app.get('/', (req, res) => {
    const data = require('./example.json')
    res.json(data);
  })

  server.listen(port, () => {
    console.log(`Example app listening at http://localhost:${port}`);
  });
```
You can see that we've enabled the Node module `fs` which allows us to interact with the filesystem, had it read in the key and cert, and created the server using the Node module `https`. These modules come with the installation of Node and do not have to be installed in `package.json`.  

Now that the server is updated, we can verify that it's handling requests using HTTPS.

### Seeing HTTPS in action
To verify that the server is using HTTPS:
* **Start the server** by running `node index.js`.
* **Make an HTTP request** by running `curl http://localhost:3000`. You should see the server respond with `Empty reply from server`. This is good!
* **Make an HTTPS request** by running `curl -k https://localhost:3000/`. You should see the familiar `{ "message": "Hello world!" }` response. Note that because we're using a self-signed certificate, we must use the `-k` flag to let curl know to ignore the self-signed certificate security warning.

Once you see the JSON response from a URL starting with `https://`, then the server has been configured to use HTTPS.