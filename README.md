# VR Blog
This is a [Jekyll](https://jekyllrb.com/) blog based off the [Chripy Theme](https://chirpy.cotes.info/). Generally speaking updating is as simple as adding a new markdown file to the post directory.  Github actions will automatically generate the site content and publish as needed when code is commited to master. The instructions below are to setup locally and view site content. 

Git on Windows will add line feeds <LF> to the scripts and break the github actions so be mindfull of that. 

## Local setup

**use WSL or Linux** 

Although you can develop on Windows most of the scripts are written for linux and simply don't play well on Windows at the moment. 

Install required packages 

```console
$ sudo apt install ruby-dev ruby-bundler coreutils fswatch zlib1g-dev build-essential 
````

Install all the ruby stuff 
```console
$ bundle install
```

Run the site locally 
```console
$ bash tools/run.sh
```

After you're done making changes push to the repo. All publishing is automated via Github Actions so you don't have to worry about that part. Code in the master branch will be generated into a site which is moved to the gh-pages branch which is used for publishing. 

## Contributing

This project welcomes contributions and suggestions.  Most contributions require you to agree to a
Contributor License Agreement (CLA) declaring that you have the right to, and actually do, grant us
the rights to use your contribution. For details, visit https://cla.opensource.microsoft.com.

When you submit a pull request, a CLA bot will automatically determine whether you need to provide
a CLA and decorate the PR appropriately (e.g., status check, comment). Simply follow the instructions
provided by the bot. You will only need to do this once across all repos using our CLA.

This project has adopted the [Microsoft Open Source Code of Conduct](https://opensource.microsoft.com/codeofconduct/).
For more information see the [Code of Conduct FAQ](https://opensource.microsoft.com/codeofconduct/faq/) or
contact [opencode@microsoft.com](mailto:opencode@microsoft.com) with any additional questions or comments.

## Trademarks

This project may contain trademarks or logos for projects, products, or services. Authorized use of Microsoft 
trademarks or logos is subject to and must follow 
[Microsoft's Trademark & Brand Guidelines](https://www.microsoft.com/en-us/legal/intellectualproperty/trademarks/usage/general).
Use of Microsoft trademarks or logos in modified versions of this project must not cause confusion or imply Microsoft sponsorship.
Any use of third-party trademarks or logos are subject to those third-party's policies.
