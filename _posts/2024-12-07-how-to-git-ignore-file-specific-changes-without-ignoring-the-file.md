---
layout: post
title: How to git ignore file specific changes without ignoring the file
date: 2024-12-07 14:32 +0100
categories:
- Git
- Security
tags:
- git
- filters
- password
- keys
- security
- sensitive
- advanced
description: Commit files that contain customizations or sensitive data, without committing the customizations nor the sensitive data itself.
---

## Introduction

When working with git, there are cases when you want to track a file but you don't want to override a part of its content,when you `git commit`, neither you want this part of its content to be overridden when you `git pull` or `git checkout` .

## Gitignore is not an option

Since you want to track this file you can't add it to the `.gitignore` file.

## Git filters is the answer

Let's assume that I have a file `src/main/resources/config/application-config.properties`, this file contains a configuration of my local application configuration.
Let's call this the property that I don't want override `assets_path`

```properties
assets_path=my_local_assets_path
dummy_property=dummy_value
```

The property value is `my_local_assets_path` but I don't want to commit it, I want to keep it on the file locally, what I want to commit instead is this value `path_to_your_assets_folder`.

### Adding filters

The first step is to add a filter to the file `.git/config` on your local repository.
It is also possible to add the file globally.

```bash
[filter "my_filter_name"]
    clean = sed 's/assets_path=.*/assets_path=path_to_your_assets_folder/g'
    smudge = sed 's/assets_path=.*/assets_path=my_local_assets_path/g'
```

* Git will run the command defined in the `clean` property when staging your changes.
* And runs the command defined in the `smudge`  property when checking out the remote changes.

### The sed command

The `sed` command is a linux command that allows searching and replacing text in files.
So basically what we are doing here is:

* Cleaning the staged filtered file when staging changes by replacing `assets_path=` and all the content after it in the same line by `assets_path=path_to_your_assets_folder`.
* Smudging the filtered file when checking it out by replacing `assets_path=path_to_your_assets_folder` and all the content after it in the same line by `assets_path=my_local_assets_path`.

This way whenever you commit the committed value will be `path_to_your_assets_folder`  and whenever you checkout the value will be `my_local_assets_path`.

### Using the filters

Now that we have our filter you need to tell git what are the files you need to filter, just add a file `.gitattributes` in the root folder of your repository.
and add this content to it.

```txt
src/main/resources/config/application-config.properties filter=my_filter_name
```

## Let's test that

Now we have everything in place, let's test it, doing the following commands in a row.

```bash
#Init the repository
mkdir -p ~/projects/blog/git-filtering
cd ~/projects/blog/git-filtering
git init

#Creating the file src/main/resources/config/application-config.properties
mkdir -p src/main/resources/config/
echo 'assets_path=my_local_assets_path\ndummy_property=dummy_value\n' >> src/main/resources/config/application-config.properties

#Adding the filter to .git/config
echo "\n[filter \"my_filter_name\"]\n\tclean = sed 's/assets_path=.*/assets_path=path_to_your_assets_folder/g'\n\tsmudge = sed 's/assets_path=.*/assets_path=my_local_assets_path/g'\n" >> .git/config
#Adding the filter to .gitattributes
echo  "\nsrc/main/resources/config/application-config.properties filter=my_filter_name\n" >> .gitattributes

# Track and stage the changes of the properties file
git add src

```

Now we have everything in place, if you check the content of your local properties file you should see:

```bash
cat src/main/resources/config/application-config.properties
```

It displays

```properties
assets_path=my_local_assets_path
dummy_property=dummy_value
```

Let's check what will be committed

```bash
git diff --staged
```

You should see something like:

```diff
diff --git a/src/main/resources/config/application-config.properties b/src/main/resources/config/application-config.properties
new file mode 100644
index 0000000..aec222b
--- /dev/null
+++ b/src/main/resources/config/application-config.properties
@@ -0,0 +1,3 @@
+assets_path=path_to_your_assets_folder
+dummy_property=dummy_value
+
```

Congratulations! As you can see, the content of the local file is different from the staged changes.

## Don't use this with sensitive data

At first sight it looks like a good idea to use the same technique to commit a file without committing its sensitive data, like passwords and api keys,  and it actually works, however I don't recommend that, because it's not meant to be used for such a thing.

Actually I already used it for `API KEYS` and when I changed the code formatter the `regular expression` used in the `sed` command stopped working, and the filter didn't work.

> If you're working with sensitive data please consider using environment variables or other alternatives.
{: .prompt-warning }

## Further reading

* [Git attributes and filters documentation](https://git-scm.com/book/ms/v2/Customizing-Git-Git-Attributes)
* [Sed command documentation](https://www.gnu.org/software/sed/manual/sed.html)
