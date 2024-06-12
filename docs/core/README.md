# Important notes

> Do **not** edit the compendium in this repository.
The compendium will be generated from the original dogu struct in the [cesapp-lib repository].
If you change the documentation here, the next `cesapp-lib` release will create a pull request to revert your changes.

## Update process of the compendium
- Make your changes in the dogu struct in the [cesapp-lib repository]
- Release `cesapp-lib`
- The release process will create a pull request in the `dogu-development-docs` repository
- Checkout new `dogu-development-docs` branch and translate your changes
- Review
- Merge

[cesapp-lib repository]: https://github.com/cloudogu/cesapp-lib/blob/develop/core/dogu_v2.go#L544