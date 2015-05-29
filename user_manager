#!/usr/bin/env python
"""
This script pulls data down from google docs and keeps system users up-to-date.

It's meant to work in tandem with a self-service google forms sheet to allow users
to keep their information up-to-date
"""

import grp
import gspread
import json
import logging
import os
import pwd
import shutil
import subprocess
from oauth2client.client import SignedJwtAssertionCredentials

# These are users that the script will cowardly refuse to edit
PROTECTED_USERS = ('ubuntu', 'root', 'wheel')

# This is the secret credentials file that was given to you by Google
CREDENTIAL_FILE = os.environ['OAUTH_CREDENTIALS_FILE']

# This is the name of the document to scan for users (the backend to the form)
DOCUMENT_TITLE = os.environ['']

def get_oauth_token(credential_file):
    """
    Returns a valid oauth2 token for using in gdocs.

    The credential_file must json-encoded and contain the following keys:

      client_email: (the long email that's tied to an api account)
      private_key:  (pem-encoded private key)

    Required args:
        credential_file: (str) the full path to the credential file that was
                         returned by google when you setup an api account.
    """
    scope = ['https://spreadsheets.google.com/feeds']
    with open(credential_file) as _file:
        creds = json.load(_file)

    return SignedJwtAssertionCredentials(creds['client_email'],
                                         creds['private_key'], scope)

def setup_skel():
    "Setup the skeleton directory used for adding new users"
    auth_dir = '/etc/skel/.ssh'
    if not os.path.isdir(auth_dir):
        logger.info("Skeleton .ssh directory does not exist. Creating it now")
        os.makedirs(auth_dir)
        os.chmod(auth_dir, 700)

def sanitize_user(username):
    """
    Verify a username is in the proper format. It's very important to guard
    against shell-injection here, since this input comes from users and is
    shelled out as root on a system.

    Therefore the name is rejected if it meeters any of the following:

        1. It contains anything but alphabetical letters
        2. Anything longer than 20 characters
        3. It belongs to a protected group of user

    If there's a problem with the user, it will return None. Else it returns the
    lower-case username.

    Required args:
        username: (str) the username to add to the system
    """
    username = str(username).lower()
    # Reject any string that isn't just alphabetical letters
    if not username.isalpha():
        logger.warning('Invalid user "%s". Username must contain letters only!',
                        username)
        return None
    # Reject any string that's longer than 10 characters
    if len(username) > 25:
        logger.warning("Username %s is too long!", username)
        return None
    if username in PROTECTED_USERS:
        logger.warning("Refusing to make adjustments for protected user: %s",
                        username)
        return None

    # This user passes muster
    return username

def add_user(username, pub_key, sudo_flag=None):
    """
    Add a user to the system. This will create the user, their home directories,
    as well as their public key. Lastly it will set the permission for the user.

    Required args:
        username: (str) the username as it should exist in the system
        pub_key: (str) public key string for authorized_keys
        sudo_flag: (bool) set to True to add this user to the sudo group
    """
    home = "/home/" + username
    pub_key_file = home + "/.ssh/authorized_keys"

    # Check if the user exists on the system before attempting to remove them
    try:
        pwd.getpwnam(username)
    except KeyError:
        logger.warning("Adding %s to system", username)
        subprocess.call("/usr/sbin/useradd -m -G madmin -s /bin/bash {0}".format(username),
                        shell=True)

    # If the sudo flag is set, add them to the sudo group. For this to work it
    # requires a change to /etc/sudoers to allow sudo without pw.
    if sudo_flag:
        if username not in grp.getgrnam('sudo').gr_mem():
            logger.warning("Granting sudo to %s", username)
            subprocess.call("usermod -a -G sudo {0}".format(username))

    # Make a few directories
    if not os.path.isdir(home):
        logger.warning("Creating home directory for %s", username)
        os.makedirs(home)
    if not os.path.isdir(home + '/.ssh'):
        logger.warning("Creating .ssh directory for %s", username)
        os.makedirs(home + '/.ssh')

    # Always update the authorized key file with the most recent key

    logger.info("Writing out public key to %s", pub_key_file)
    with open(pub_key_file, "w") as _file:
        _file.write(pub_key)

    # Make sure all of the permissions are set
    logger.info("Setting home directory permissions for %s", username)
    subprocess.call("/bin/chown -R {0}:{0} {1}".format(username, home), shell=True)

def purge_user(username):
    """
    Purge a user from the system. This will delete the user as well as
    recursively delete their home directory.

    Required args:
        username: (str) the user to remove from the system and /home/
    """
    home = "/home/{0}".format(username)

    try:
        pwd.getpwnam(username)
    except KeyError:
        logger.info("User %s does not exist. Not purging from system.", username)
    else:
        logger.warning("Deleting %s from system", username)
        subprocess.call("/usr/sbin/userdel -f -r {0}".format(username), shell=True)

    if os.path.isdir(home):
        logger.warning("Removing home directory %s", home)
        shutil.rmtree(home, ignore_errors=True)

if __name__ == '__main__':

    logging.basicConfig(level=logging.WARNING, format='[%(levelname)s] %(message)s')
    logger = logging.getLogger()

    # Do some basic setup
    setup_skel()

    gc = gspread.authorize(get_oauth_token(CREDENTIAL_FILE))
    users = gc.open(DOCUMENT_TITLE).sheet1

    if not users:
        raise Exception("Unable to find any users in the system! Make sure that"
                        " the google document is named correctly and shared "
                        "with the user that has the credentials file")

    # Seen users is a set that holds users we've seen before. We keep track of
    # them within this set so that one user can't overwrite another one by using
    # the same username
    seen_users = set()

    # Iterate through all of the users and add them to the system
    for row in users.get_all_records():

        # We consider this variable dangerous until proven otherwise. We need to
        # guard against shell-injection attacks here
        dangerous_username = row['SSH username']

        email = row['Username']
        public_key = row['SSH Public Key']
        purge_flag = row.get('purge')

        # Make sure their username is valid. This comes back as None we'll
        # assume the username is invalid
        sane_user = sanitize_user(dangerous_username)
        if not sane_user:
            logger.warning('Skipping user "%s" for email %s', dangerous_username,
                           email)
            continue

        # Keep two users from using the same ssh username and overwriting each-
        # other. This is also a security risk.
        if sane_user in seen_users:
            logger.warning('Skipping %s\'s user "%s". Another user is using the '
                           'same same ssh username.', email, sane_user)
            continue
        else:
            seen_users.add(sane_user)

        # Purge an existing user
        if purge_flag == "x":
            purge_user(sane_user)
            continue

        # Add the user (and potentially cleanup permissions)
        add_user(sane_user, public_key)
