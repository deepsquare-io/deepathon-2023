# Smartguy

Simple web app exposed with Nginx as a reverse proxy using a Bore tunnel. Will show different colors depending on the role you will connect to.

Note : Persistence is made with a mariaDB which is a bad practice with DeepSquare since once container is shut down, data is lost. This project was more like a demo to see what simple services could look like using DeepSquare limitations.


## How-to-use

Just submit the job:

```shell
dps submit --watch --credits 1000 --job-name simple-web-app job.smartguy.yaml
```

Now copy paste your "deepsquare.run" address in a browser, try to log in with the following credentials "admin/adminpass", "editor"/"editorpass" or "viewer"/"viewerpass".
Observe the color change depending on the user you are connected to ! This is a really basic example of permissions.

## Improvements : 
For such app, you could for example :
1. make it stateless
2. use a cloud provider DB solution (e.g. Amazon RDS, DynamoDB)
3. use AWS SDK for S3 to interact with S3 buckets to perform operations using S3 APIs
4. use your own S3 solution (e.g. Ceph)

Assuming you have a JSON file with user data structured like this :

```shell
[
    {
        "username": "admin",
        "password": "adminpass",
        "role": "Administrator",
        "color": "red"
    },
    {
        "username": "editor",
        "password": "editorpass",
        "role": "Editor",
        "color": "green"
    },
    {
        "username": "viewer",
        "password": "viewerpass",
        "role": "Viewer",
        "color": "blue"
    }
]
```

You could with the following Node.js code do the exact same application retrieving data from S3 and parse it to get user data :

```shell
const AWS = require('aws-sdk');
const s3 = new AWS.S3({
    accessKeyId: process.env.AWS_ACCESS_KEY_ID,
    secretAccessKey: process.env.AWS_SECRET_ACCESS_KEY,
    region: process.env.AWS_REGION
});

const bucketName = 'your-s3-bucket-name';
const usersFileKey = 'users.json';

// Function to get user data from S3
function getUsersFromS3(callback) {
    const params = {
        Bucket: bucketName,
        Key: usersFileKey
    };

    s3.getObject(params, (err, data) => {
        if (err) {
            console.error("Error fetching users from S3:", err);
            callback(err);
            return;
        }
        try {
            const users = JSON.parse(data.Body.toString());
            callback(null, users);
        } catch (jsonErr) {
            console.error("Error parsing user data:", jsonErr);
            callback(jsonErr);
        }
    });
}

// Express app setup
const express = require('express');
const app = express();
app.use(express.urlencoded({ extended: true }));

app.get('/', (req, res) => {
    res.send('<form method="post">Username: <input type="text" name="username"/><br/>Password: <input type="password" name="password"/><br/><input type="submit" value="Login"/></form>');
});

app.post('/', (req, res) => {
    const { username, password } = req.body;

    getUsersFromS3((err, users) => {
        if (err) {
            res.send("Error retrieving user data");
            return;
        }

        const user = users.find(u => u.username === username && u.password === password);
        if (user) {
            res.send(`<div style='color: ${user.color};'>Logged in as ${user.role}</div>`);
        } else {
            res.send('Invalid username or password');
        }
    });
});

const port = process.env.PORT || 3000;
app.listen(port, () => {
    console.log(`Server running at http://localhost:${port}`);
});
```
