github-email-notifications
==========================

Better email notifications from github with clear, concise, and readable
messages.

[![Build Status](https://travis-ci.org/chapel-lang/github-email-notifications.svg?branch=master)](https://travis-ci.org/chapel-lang/github-email-notifications) [![Coverage Status](https://coveralls.io/repos/chapel-lang/github-email-notifications/badge.svg?branch=master)](https://coveralls.io/r/chapel-lang/github-email-notifications?branch=master)

Simple python web application for Heroku that accepts github webhooks for
["push" events][push_events] and generates a clear, concise, and readable email
message.

It is designed to meet the [Chapel][chapel] team's needs, as the email hook
provided by github is rather noisy and it looks unlikely to change. The Chapel
project uses the [feature branch workflow][fb_workflow], and always includes a
nice summary in the merge commit messages. So, the merge message (actually the
head commit in the push event) is the only one included in the email.

Also be aware of this issue (https://github.com/Cray/chapel-private/issues/3799)
"Identifying and Implement new solution for Heroku Commit Emailer" (so you may
want to check there to see if the info is out-of-date).

[chapel]: http://chapel-lang.org/
[fb_workflow]: https://www.atlassian.com/git/tutorials/comparing-workflows/feature-branch-workflow/
[push_events]: https://developer.github.com/v3/activity/events/types/#pushevent

Heroku Setup
------------

Create the app, enable papertrail to record logs, and set the configuration
variables.  Note: if you have more than one Heroku app associated with your
login, most heroku command lines after "create" (see below) should include 
"-a <app_name>" on the end of the command line.  You can also do everything 
from the Heroku web page, https://dashboard.heroku.com/apps/<app_name>, 
instead of the command line- however, that's not shown here.

```bash
# heroku login
  # Create your own login, or use 
  # chapel_github@cray.com + password from Keeppass.
  # After the first time, the heroku login persists somewhere for a while
heroku create <app_name>
heroku addons:add papertrail
heroku config:set GITHUB_COMMIT_EMAILER_SEND_FROM_AUTHOR=<true>
heroku config:set GITHUB_COMMIT_EMAILER_SENDER=<sender_email>
heroku config:set GITHUB_COMMIT_EMAILER_RECIPIENT=<recipient_email>
heroku config:set GITHUB_COMMIT_EMAILER_SECRET=<the_secret>
```

An optional "RECIPIENT_CC" e-mail address may be given, in which case the messages
are Cc'd to that address. 

```bash
heroku config:set GITHUB_COMMIT_EMAILER_RECIPIENT_CC=<Cc_email>
```

If `GITHUB_COMMIT_EMAILER_SEND_FROM_AUTHOR` is set (to any value), the pusher
name and email combination will be used as the "From" address instead of the
configured sender value. If a reply-to is configured, see below, that will be
added regardless of this setting.

Optionally, a reply-to address can be configured with the following config. If
not set, no reply-to header is set so the sender address will be used as reply
address.

```bash
heroku config:set GITHUB_COMMIT_EMAILER_REPLY_TO=<reply_to_email>
```

Optionally, an "Approved" header value can be configured. The Approved header
automatically approves the messages for a read-only or moderated mailing list.

```bash
heroku config:set GITHUB_COMMIT_EMAILER_APPROVED_HEADER=<approved_header>
```

SendGrid Setup
--------------

Enable addon, and disable the plain text to html conversion:

```bash
heroku addons:add sendgrid
heroku addons:open sendgrid
```

* Go to "Settings"-> "Mail Settings".
* Activate "Plain Content - Convert your plain text emails to HTML."
  (ie, "Turn on if you don't want to convert your plain text email to HTML" -
  we want plain content, not HTML)

Rollbar Setup
-------------

Rollbar provides error tracking, in case anything unexpected happens. Enable
the addon and optionally set the environment name.

```bash
heroku addons:add rollbar
```

Optionally, set the environment name for rollbar. This is probably only
necessary if you have multiple environment configured to use rollbar.

```bash
heroku config:set GITHUB_COMMIT_EMAILER_ROLLBAR_ENV=<env_name>
```

Deploy the Heroku App
---------------------

I found this easier to do from the Heroku web page,
https://dashboard.heroku.com/apps/<app_name>/deploy/github. (ie, Heroku 
dashboard for app <app_name>, "Deploy" tab, deployment method "GitHub"). 
If you "connect" the Heroku app to this GitHub repo, you can Deploy right
from there. Because "connecting" means allowing auto authentication
to GitHub from Heroku, I preferred not to Deploy as chapel_github@cray.com. 
Instead, I deployed from my personal Heroku account, which I added to the
Heroku app as a "collaborator" (see "Access" tab). 

GitHub Setup
------------

Add webhook to repo to use this emailer. Be sure to set the secret to the value
of `GITHUB_COMMIT_EMAILER_SECRET` (you are free to generate any string). 
The webhook URL is `<heroku_url>/commit-email` and it must send "push" events in
JSON format. Show the heroku app url with:

```bash
heroku domains
```

Development
-----------

To develop and test locally, install the [Heroku Toolbelt][0], python
dependencies, create a `.env` file, and use `heroku local:start` to run the app.

* Install python dependencies (assuming virtualenvwrapper is available):

```bash
mkvirtualenv github-email-notifications
# if not found, source virtualenvwrapper.sh and retry
pip install -r requirements.txt
```

* Create `.env` file with chapel config values:

```
GITHUB_COMMIT_EMAILER_SENDER=<email>
GITHUB_COMMIT_EMAILER_RECIPIENT=<email>
GITHUB_COMMIT_EMAILER_SECRET=<the_secret>
ROLLBAR_ACCESS_TOKEN=<rollbar_token>
SENDGRID_PASSWORD=<sendgrid_password>
SENDGRID_USERNAME=<sendgrid_user>
```

To use the same values configured in heroku:

```bash
# heroku login
  # chapel_github@cray.com + password from Keeppass
  # after the first time, the heroku login persists somewhere for a while
heroku config --shell > .env
```

* Run the app, which opens at `http://localhost:5000`:

```bash
heroku local:start
```

* Send a test request:

```bash
curl -vs -X POST \
  'http://localhost:5000/commit-email' \
  -H 'x-github-event: push' \
  -H 'content-type: application/json' \
  -d '{"deleted": false,
    "ref": "refs/heads/master",
    "compare": "http://compare.me",
    "repository": {"full_name": "test/it"},
    "head_commit": {
      "id": "mysha1here",
      "message": "This is my message\nwith a break!",
      "added": ["index.html"],
      "removed": ["removed.it"],
      "modified": ["stuff", "was", "done"]},
    "pusher": {
      "name": "awallace",
      "email": "testing@github-email-notification.info"}}'
```

Note: the above test will not actually send an email, because
```WARNING:root:Invalid signature, skipping request```

* Install test dependencies and run the unittests.

```bash
pip install -r test-requirements.txt
tox
tox -e flake8
tox -e coverage
```

Update: 2017-10-19
------------------

Readme is updated as deployed to Heroku app chapel-commit-emailer2
(https://dashboard.heroku.com/apps/chapel-commit-emailer2)

[0]: https://toolbelt.heroku.com/
