Warning
=======

Branch protection is not enabled and GitHub Actions has the
ability to write to the repository. Any action or code executed
during the build, even code in third party dependencies called
during testing, could subvert the integrity of your project.

See [Disabling or limiting GitHub Actions for a repository](https://docs.github.com/en/actions/reference/environment-variables#default-environment-variables) for more information on how to protect you repository.