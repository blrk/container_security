### Running a containerized application in HTTPS
Objective: This excercise eduacte you on running a containerized application in HTTPS, how to set up SSL/TLS certificates, configure your application to use these certificates, and ensure your container environment supports HTTPS. 
* Install Certbot
```bash
sudo dnf install -y augeas-libs
sudo python3 -m venv /opt/certbot/
sudo /opt/certbot/bin/pip install --upgrade pip
sudo /opt/certbot/bin/pip install certbot
```
* Create a myapp directory and navigate into that. 
```bash
mkdir ~/myapp; cd ~/myapp
```
* Creating Self-Signed Certificates (for local development):
```bash
openssl req -x509 -newkey rsa:4096 -keyout key.pem -out cert.pem -days 365

.....+...+...............+...+............+...+........+.......+..+...+...............+.........+....+..+...............+.......+........+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++*....+....+...+...+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++*...............+......+........+......+......+...+.+......+...+.........+...+....................+......................+...+....................+...+....+............+.....+.+.....+..........+..+...+.......+..+......+....+......+.....+.......+.................+.........+.+......+.....+..........+...+.........+.....+.+..+....+...+..+.........+.........+.......+.....+...+.............+........+.......+..+......+.+..................+.....+...............+.+......+......+...+...........+...+.+...........+....+.....+......+................+...+.............................+......+.+........+......+.......+...+...........+...+...+...+.........+.......+...+...+...+.....+.......+...+..+.........+....+.....+.+........+...+....+..............+.+.....+...+...+...+....+...+........+...............+....+.................+...+.+..+...................+........+.+...........+...+.............+........+......+.........+......+.+............+...........+....+...+...+............+..+.+.................+....+.....+.+..+......+......+.+.................+.+......+.........+......+..................+.....+.+......+.....+....+......+.....+......+...+.+...+........+............+.........................+.....+...+..........+..+............+...+.+......+.....+...............+......+...+....+.....+...+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
...............+........+....+..+...+.+......+...+.....+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++*.+...+....+..............+....+.....+.......+......+.....+...+.+...+..+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++*.+......+.+.....+.+...............+........+.....................+...+.+..+............+................+.....+.+.........+......+..............+......+.+...+..+.......+...............+....................+..........+..+..........+.....+...+...+.......+........+......................+...+.........+...+.....+.............+........+......+.+...+.....+...............................+...........+.........+......+....+...............+......+..............+..........+......+..+.+...........+......+....+...+...+...+............+.....+.......+............+...+......+..+............+...............+......+......+...+...+......................+.....+......................+........+.+..+.......+......+......+..............+.........+.......+...........+...+................+...........+..................+.......+...+.....+.+.......................+......+......+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
Enter PEM pass phrase:
Verifying - Enter PEM pass phrase:
-----
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [XX]:IN
State or Province Name (full name) []:Tamilnadu
Locality Name (eg, city) [Default City]:Chennai
Organization Name (eg, company) [Default Company Ltd]:SCB
Organizational Unit Name (eg, section) []:training 
Common Name (eg, your name or your server's hostname) []:myapp.sc.com
Email Address []:xxx.sc.com
```
* Ensure the key file does not require a passphrase. If it does, you need to remove the passphrase or handle it in your code.
```bash
openssl rsa -in key.pem -out key_nopass.pem
mv key_nopass.pem key.pem
```
* Create package.json
```bash
vi package.json

{
  "name": "myapp",
  "version": "1.0.0",
  "description": "A simple Node.js app running in HTTPS mode in Docker",
  "main": "app.js",
  "scripts": {
    "start": "node app.js"
  },
  "dependencies": {
    "express": "^4.17.1"
  }
}
```
* Create app.js
```bash
vi app.js

const fs = require('fs');
const https = require('https');
const express = require('express');

const app = express();

app.get('/', (req, res) => {
  res.send('Hello, HTTPS! I am  from container security');
});

const options = {
  key: fs.readFileSync('/etc/ssl/private/key.pem'),
  cert: fs.readFileSync('/etc/ssl/certs/cert.pem')
};

https.createServer(options, app).listen(443, () => {
  console.log('Server running on https://localhost:443');
});
```

* Create the Dockerfile
```bash
FROM node:14

# Create app directory
WORKDIR /usr/src/app

# Install app dependencies
COPY package.json ./
RUN npm install

# Bundle app source
COPY app.js .

# Copy SSL certificates
COPY cert.pem /etc/ssl/certs/cert.pem
COPY key.pem /etc/ssl/private/key.pem

EXPOSE 443
CMD [ "node", "app.js" ]
```
* Build and Run the Docker Container
```bash
docker build -t myapp .
```
```bash
docker run -itd --name myapp -p 443:443  myapp
21c86b016b17dd745beaaef10c3513792e03813d5cba0c74301304bab2418069
```
* List the running containers
```bash
docker ps
CONTAINER ID   IMAGE     COMMAND                  CREATED          STATUS          PORTS                                   NAMES
21c86b016b17   myapp     "docker-entrypoint.s…"   12 minutes ago   Up 12 minutes   0.0.0.0:443->443/tcp, :::443->443/tcp   myapp
```
* Access the app
```bash
curl -k https://localhost

Hello, HTTPS! I am  from container security
```