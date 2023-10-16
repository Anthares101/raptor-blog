# Raptor Blog

Just a little personal blog where I will publish cybersecurity stuff.

The page is published here: https://blog.anthares101.com/

## How to deploy locally

Install the requirements:
```bash
sudo apt install hugo

# Inside the project folder to get the theme
git submodule init
git submodule update
```

Now you can start the server from the project root directory:
```bash
# The -D option make sure drafts are also published (We want that while editing)
hugo server -D
```
Now you should be able to see the server running in http://localhost:1313/
