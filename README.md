git-eventc
==========

git-eventc is a small project aiming to bridge Git repositories to eventd for commit notification.
<br />
This project is only useful in a working eventd environment.
Most people will need the eventd `im` plugin to act as an IRC commit bot.


Events
------

git-eventc will provide events in the `scm` event group: `commit`, `commit-group`, `branch-created`, `branch-deleted`, `tag-created`, `tag-deleted`.
<br />
Here is the list of common data provided by all events:

* `repository-name`: The name of the repository
* `repository-url`: The URL of the repository
* `project-group`: The project group name (if set)
* `project`: The project name, defaults to `repository-name`
* `pusher-name`: The name of the pusher
* `branch`: The updated branch name (not for `tag-` events, and the related `push` event)
* `url`: An URL to see the change online (not for `-deleted` events)


### `commit`

This event correspond to a single commit.
<br />
Here is the list of provided data:

* `id`: The commit id (short version, see `--help`)
* `subject`: The commit subject (first line of message)
* `message`: The commit message (with subject and footer tags stripped, only if not empty)
* `full-message`: The full commit message (verbatim)
* `author-name`: The name of the author
* `author-email`: The email of the author
* `author-username`: The username of the author (if available)
* `files`: The list (as a string) of modified files, with some basic prefix detection
    <br />
    The `post-receive` hook also detects file renames and copies if asked so.


### `commit-group`

This event correspond to a group of commit.
It will be generated if a push is adding a number of commits above a specified threshold (see `--help`).
<br />
Here is the list of provided data:

* `size`: The number of commits in this push


### `branch-created` and `branch-deleted`

This event correspond to the creation/deletion of a branch.


### `tag-created` and `tag-deleted`

This event correspond to the creation/deletion of a tag.
<br />
Here is the list of provided data:

* `previous-tag`: The latest tag in this tag history tree (for `tag-created` only)


### `push`

This event correspond to a push.
It will be generated after a set of `commit` events, or any of other events events.
<br />
This event is useful for mirroring purpose.


### Example event file

(See eventd configuration for further information.)

For a `commit` event:

    [Event]
    Category = scm
    Name = commit
    [IMAccount freenode]
    Message = ${project-group}/^B${project}^O/^C07${branch}^O: ^C03${author-name}^O * ${id}: ${message} ^C05${url}^O ^C14${files}^0
    Channels = #test;

For a `commit-group` event:

    [Event]
    Category = scm
    Name = commit-group
    [IMAccount freenode]
    Message = ${project-group}/^B${project}^O/^C07${branch}^O: ^C03${pusher-name}^O pushed ${size} commits ^C05${url}^O
    Channels = #test;

For a `branch-created` event:

    [Event]
    Category = scm
    Name = branch-created
    [IMAccount freenode]
    Message = ${project-group}/^B${project}^O/^C07${branch}^O: ^C03${pusher-name}^O branch created ^C05${url}^O
    Channels = #test;

For a `branch-deleted` event:

    [Event]
    Category = scm
    Name = branch-deleted
    [IMAccount freenode]
    Message = ${project-group}/^B${project}^O/^C07${branch}^O: ^C03${pusher-name}^O branch deleted ^C05${url}^O
    Channels = #test;



Executables
-----------


You can specify configuration either directly on the command-line, or in a file `~/.config/git-eventc.conf`, in the `key=value` format.
All keys must be in a `[git-eventc]` group and use the same name as their command-line argument.

### git-eventc-post-receive

git-eventc-post-receive is a Git post-receive hook.
See `--help` output for basic configuration.
<br />
You can use it directly as a post-receive hook or in a wrapper script. Please make sure `stdin` is fed correctly.

It will use configuration directly from Git.
You should configure most of them in your system configuration (`/etc/gitconfig`).
<br />
Configuration value names are prefixed by `git-eventc.`. Here is the list of used values:

* `project-group`: used as `project-group`
* `project`: used as `project`, defaults to `repository-name`
* `repository`: used as `repository-name` (not meaningful in system configuration)
* Several URL template strings:
  all of them have the `${repository-name}` token.
    * `repository-url`: URL template for the repository:
        * Examples: `http://cgit.example.com/${repository-name}` or `http://gitweb.example.com/?p=${repository-name}.git`
    * `branch-url`: URL template for a branch, available token:
        * `${branch}`: the name of the branch
        * Examples: `http://cgit.example.com/${repository-name}/log/?h=${branch}` or `http://gitweb.example.com/?p=${repository-name}.git;a=shortlog;h=refs/heads/${branch}`
    * `commit-url`: URL template for a single commit, available token:
        * `${commit}`: the commit id
        * Examples: `http://cgit.example.com/${repository-name}/commit/?id=${commit}` or `http://gitweb.example.com/?p=${repository-name}.git;a=commitdiff;h=${commit}`
    * `tag-url`: URL template for a tag, available token:
        * `${tag}`: the tag name
        * Examples: `http://cgit.example.com/${repository-name}/commit/?id=${tag}` or `http://gitweb.example.com/?p=${repository-name}.git;a=commitdiff;h=${tag}`
    * `diff-url`: URL template for a diff between two commits, available tokens:
        * `${old-commit}`: the old commit id
        * `${new-commit}`: the new commit id
        * Examples: `http://cgit.example.com/${repository-name}/diff/?id2=${old-commit}&id=${new-commit}` or `http://gitweb.example.com/?p=${repository-name}.git;a=commitdiff;hp=${old-commit};h=${new-commit}`

It also has support for Gitolite environment variables:

* `GL_USER`: used as `pusher-name`
* `GL_REPO`: used as `repository-name`


### git-eventc-webhook

git-eventc-webhook is a tiny daemon that will listen HTTP POST based hook.
These are provided by many Git host providers.
<br />
See `--help` output for its configuration.

Just run it or your server and point the WebHook to it.
You can use the proxy support of your favorite web server if you prefer.
<br />
Direct TLS/SSL support is avaible.

git-eventc-webhook will split the URL path in two:

* first part will be used as `project-group`
* second part (may contain slashes) will be used as `project`
    <br />
    The second part is optional and will default to `repository-name`

Here is the list of supported services.

* GitHub

Example URLs:

    http://example.com:8080/TestProjectGroup
    https://example.com:8080/TestProjectGroup/TestProject
    https://example.com/webhook/TestProjectGroup/TestProject (behind Apache ProxyPass)

#### Secrets

git-eventc-webhook has secret support. In your GitHub WebHook configuration, you can specify a secret.
This secret will be used to compute a signature of the hook payload, which is sent in the request header.
git-eventc-webhook will compute the signature and compare it with the one in the request.

To specify secrets, you must use a configuration file. Here is the format:

* The group name is `[webhook-secrets]`.
* Each key is a project group name
* Each value is the corresponding secret

Example:

    [webhook-secrets]
    Group1=secret
    Group2=secret
    Group3=other-secret
