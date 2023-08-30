---
title: "Update Github Profile Readme Dynamically using a Github Workflow"
date: 2023-08-29T20:44:26-04:00
draft: false
metathumbnail: "/readme_action/github-mark.png"
description: "Github introduced the option to add profile readme, which is a
special repository that is used to display a readme on your profile page. This
is an action that updates the readme dynamically with recent blog posts using a
simple python script, and a scheduled workflow on github."
tags: ["github", "python", "action", "CI/CD", "workflow"]
categories: ["software", "python", "github"]
keywords: ["github", "python", "action", "CI/CD", "workflow", "CI/CD",
  "workflow"]
---

**TL;DR**: Update your Github Profile Readme dynamically with recent blog posts.
There's an example ([profile](https://github.com/heathhenley) and
[code](https://github.com/heathhenley/heathhenley)).

## Github Profile Readme
Github introduced the option to a add profile readme, which is a special
repository that is used to display a readme on your profile page. I wanted to
post links to my recent blog posts there, so I wrote a simple python script to
check for the latest X posts, and update the readme with links to them. Then I
used a scheduled workflow to run the script every day so my new posts are always
displayed.

## The Python Script
The python script is pretty simple. It takes in the url of the xml feed for the
blog, so on mine for example, it is [here](https://heathhenley.com/index.xml).
Then it parses out the title and url of the latest X posts and replases a
placeholder in the readme with the links. You can see the full script in the
repo [here](https://github.com/heathhenley/heathhenley/blob/main/scrape_blogs.py), but here's the main part:

```python
def main():
    blog_feed_url = os.getenv('BLOG_FEED_URL', None)
    number_of_posts = os.getenv('NUMBER_OF_POSTS', 10)
    readme_name = os.getenv('README_TEMPLATE_NAME', 'README_template.md')
    if not blog_feed_url:
        print('Please set the BLOG_FEED_URL environment variable')
        return
    print('Scraping blog feed from: {}'.format(blog_feed_url))
    print('Number of posts: {}'.format(number_of_posts))
    url_title_map = xml_to_url(
        get_blog_feed_xml(blog_feed_url), number_of_posts)
    add_to_readme(url_title_map, readme_name)
```

The `BLOG_FEED_URL` needs to be set in the Github repository settings, I set it
as a variable (under 'reponame/settings/variables/actions' on github). The
other variables (`NUMBER_OF_POSTS` and `README_TEMPLATE_NAME`) are optional, but
can be set in the same way. The `README_TEMPLATE_NAME` is the name of the readme
that you want to update, it should contain the placeholder text `BLOG_POST_LIST`
to replace with the list of blog posts.

## The Github Workflow
Now that we have the script to update the readme, we need to run it on a regular
schedule, and then commit the updated reamdme to the repo automatically. Here's
the [Workflow](https://github.com/heathhenley/heathhenley/blob/main/.github/workflows/scrape_blogs.yaml) that I used to do that:

```yaml
name: "Get latest blogs"
on:
  schedule:
  - cron: "0 0 * * *"
jobs:
  scrape_blog:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Set up python 3.10
        uses: actions/setup-python@v4
        with:
          python-version: "3.10"
          cache: "pip"
      - name: Install deps
        run: pip install -r requirements.txt
      - name: Run blog scraper
        env:
          BLOG_FEED_URL: ${{ vars.BLOG_FEED_URL }}
        run: python scrape_blogs.py
      - name: Update commit author
        run: |
          git config user.name 'github-actions[bot]'
          git config user.email 'github-actions[bot]@users.noreply.github.com'
      - name: Add and commit readme
        run: |
          git add README.md
          git commit -m "Automatic readme update (blog scraper)" || exit 0
          git push origin main
```

We can see the `Run blog scraper` step passes in the environment variables from 
the repo variables. That step runs the python script above, which gets the
latest blogs and copies them into the readme. Then we just need to commit the
result and push it to the repo. The ` || exit 0` on the commit step is so that
if there is nothing to commit (no new blog posts), the workflow doesn't fail.

## That's all
That's all there is to it. Now you can have a dynamic readme on your Github
Profile, you may need to make some changes to the script to fit your specific
blog url, but this is the idea.