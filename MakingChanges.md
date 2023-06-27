# Making Changes

In this section we will be walking through how to make a change. Some of the instructions you will have to do in GitHub, some on your laptop.

## Some Basics

First some general guidelines on what makes a good change.

- Only minimum change sets will be accepted. The change must be focused and purposeful; the change should match the original request; the commit description must match the change. No surprises. You can do house-keeping or refactoring of code but that must be under a separate and well defined change request.

- The only coding standard is that the code must all look the same. It has to look like one person wrote the whole thing.

- Testing code coverage must be equal or better for each change request. Make sure you add the necessary unit tests.

There are always better ways to do something. This application will be used by 10-100 people only. I want it to be clean, functional, and help us get to root cause quickly, but it doesnt have to be slick or well engineered - not the first iteration anyway.

With that said, to discuss how you can make a change we will use `splogparser` as an example but the same will be true for all tools.

## Git Clone

From a cmd.exe navigate to the folder where you hold all your Git repositories (e.g. `C:\Git`). Execute `git clone` using clone argument provided by Git (see image below):

![image](./images/gitclone.png)

For example:

```text
C:\Git\git clone https://github.com/HyosungTools/splogparser.git
```

This should create the folder `C:\Git\splogparser` and within will be mainline source.

Notice GitHub now provides Codespaces. Codespaces are cloud based development environments. I've heard about this but never used it. I dont think you have to develop on your laptop at all anymore. That said, for these examples I will assume you are.

## Issues

All changes start with an GitHub Issue.

![image](./images/gitissue.png)

An issue is just a suggested change; it can be a feature enhacement or bug fix. If you assign your self an issue make sure you:

- set yourself as assigned
- label it correctly

Most important - create a branch for the work. All issue work must be on a separate branch. Also note only I am allowed to merge into main, so do all your Issue work on a separate branch.

## Branch

As mentioned above all work happens on a branch. Do this in GitHub. When viewing an issue on the right hand side under the word 'Development' is the option to 'create a branch'.

![image](./images/gitbranch.png)

Selecting this option displays the Create Branch dialog.  

Having created the branch in GitHub you now need to refresh your local repository. Using splogparser as an example, execute the following git commands:

```text
C:\Git\splogparser\git fetch origin
C:\Git\splogparser\git checkout <branch name>
```

The first command will pull down from GitHub all changes to the repository including the branch you just created. The second command will move you onto that branch for work.

Do all your work on that branch.

## Work

When I work, I always have a cmd.exe open in my git repository. To intialize it I run these commands:

```text
C:\Git\splogparser> build\setenv.cmd
C:\Git\splogparser> set path=%path%;C:\Program Files (x86)\Microsoft Visual Studio\2022\BuildTools\Common7\IDE\CommonExtensions\Microsoft\TestWindow
```

And then to build the application and run the unit tests I execute: 

```text
C:\Git\splogparser> msbuild src\\splogparser.sln /m:1 /t:Clean,Build /p:Configuration="Release" /p:Platform="Any CPU" /nodeReuse:false
C:\Git\splogparser> msbuild src\\splogparser.sln /t:RunUnitTests /p:Configuration="Release" /p:Platform="Any CPU"
```

I do this because this is what happens on the build server; these commands are right out of the build.yml. I want to make sure that when I create a pull request the tests are all going to pass.

I also have a seconds cmd.exe open in a folder containing sample logs, for example: `C:\Documents\Bugs\NHSWS-7624>`. In the first cmd.exe I can make sure the application builds and all unit tests run; in the second cmd.exe i can run what I just built to see if its doing what I expect it to do. 

I use Visual Studio as my editor but I build, run and test using cmd.exe. 

## Commit

When you are finished your work you can commit your changes. I want to use ![conventional commits]([https://www.conventionalcommits.org/en/v1.0.0/). You should use conventional commits in your commit messages but I think its real impact is in merge commits.  

## Pull Request

Once you have pushed your commit back up to GitHub, back in GitHub you can create a pull request. When you create a Pull Request all unit tests are run; that's the first text. The second test is a code review. Again the code review is going to look for these things:

- is this a minimum change?
- does this code look like the rest of the code?
- are there unit tests?

## Merge

All merges are done by me. When a Pull Request is merged into mainline it also runs all unit tests.

## Merge Conflict

I'm going to kick all merge conflicts back to the developer to fix. Development is pretty isolated to Views. The only merge conflict I think we will see will be in the sln file. 

To resolve a merge conflict I think you need to do the following commands in Git:

```text
git checkout main
git pull
git checkout <branch>
git rebase -i master
```

Basically reproduce the merge conflict in your local repo by refreshing main and then refreshing your branch from main. Make any necessary changes and commit. 

## Delete Branch

When the merge is complete the default action will be to delete the branch.
