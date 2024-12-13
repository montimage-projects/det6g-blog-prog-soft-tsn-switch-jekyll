These are the source files of the [DETERMINISTIC6G blog](https://blog.deterministic6g.eu/).

The blog is build as a static web page using [Jekyll](https://jekyllrb.com/).

The release version of the blog (to be deployed on the public blog server) is always in the main branch.

New blog entries are started as branches, and after the review phase and final revision, the final version is merged with the main branch.

# Prerequisites

The blog is created using Jekyll.
To install Jekyll on Debian/Ubuntu, follow the instructions on this web page: [Installing Jekyll on Debian/Ubuntu](https://jekyllrb.com/docs/installation/ubuntu/)

Check the [Jekyll](https://jekyllrb.com/) web page for installation instructions for other platforms.

# Creating the blog and deploying on public web server

Clone the main branch of the repository, which always contains the latest release version of the blog:

```
$ git clone https://deterministic6g.informatik.uni-stuttgart.de/d6g/blog.git
$ cd blog
```

Be sure that the domain and URL parameters are set correctly in the file `_config.yml`:

```
domain: blog.deterministic6g.eu
url: https://blog.deterministic6g.eu
```

The parameter `baseurl` must not be set since the blog is hosted directly under the URL https://blog.deterministic6g.eu.

Create the static web page from the root directory of the repository:

```
$ bundle exec jekyll build
```

The directory `_site` now contains all files of the static web page.
Copy this directory to the root directory of the public web server.

# Creating new blog entries

If you are unsure what to do or if you do not want to write your blog posts as Markdown, you can simply write your blog as Word file, and after the internal review provide the Word document and all images in jpg or png format to Frank Dürr.
A student assistant will take care to convert the blog entry to Markdown and upload it to this Git repository.

If you want to write your blog entry as Markdown, follow these steps:

New blog entries are started in new branches and later merged with the main branch after the entry is final (after review and final revision).
For instance, to start a new blog entry on the topic Network Delay Emulator, first create a new branch (e.g. `post_delayemu`) from the main branch:

```
$ git clone https://deterministic6g.informatik.uni-stuttgart.de/d6g/blog.git
$ cd blog
$ git checkout -b post_delayemu
```

Write your blog entry:

* Create a markdown (`*.md`) file in the folder `_posts`. The file name should start with the date, e.g., `2024-10-26-network_delay_emulator.md`
* Add your text to the markdown file and copy images to the folder `assets/images/`.
* You might want to have a look at an existing blog entry in `_posts` to see how to format text, reference images, add author information, etc.

Commit you changes and push them to the branch:

```
$ git push origin post_delayemu
```

To check your post and for internal review, upload your post to an internal web server.
You can edit the following parameters in the `_config.yml` file depending on the URL and path of your web server:

```
domain: blog.myserver.de
url: https://blog.myserver.de
baseurl: /path/to/blog
```

Create the static web page from the root directory of the repository:

```
$ bundle exec jekyll build
```

The directory `_site` now contains all files of the static web page.
Copy this directory to the root directory of the public web server.

When the blog post is final, merge it with the main branch.
From the root directory of the repository, execute the following commands (replace `post_delayemu` by your branch name):

```
$ git checkout main
$ git merge post_delayemu
$ git push origin main
```