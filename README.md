# Getting Started

1. Register a GitHub account if you don't have one and confirm your e-mail (required!)
1. Click the "Use this template" button in the top right.
1. On the "New repository" page, enter your repository name as "[your GitHub username].github.io", which will also be your website's URL.
1. Set site-wide configuration and add your content.
1. Upload any files (like PDFs, .zip files, etc.) to the `files/` directory. They will appear at https://[your GitHub username].github.io/files/example.pdf.
1. Check status by going to the repository settings, in the "GitHub pages" section
1. (Optional) Use the Jupyter notebooks or python scripts in the `markdown_generator` folder to generate markdown files for publications and talks from a TSV file.

See more info at https://academicpages.github.io/

## Running locally

When you are initially working your website, it is very useful to be able to preview the changes locally before pushing them to GitHub. To work locally you will need to:

1. Clone the repository and made updates as detailed above.
1. Make sure you have ruby-dev, bundler, and nodejs installed
    
    On most Linux distribution and [Windows Subsystem Linux](https://learn.microsoft.com/en-us/windows/wsl/about) the command is:
    ```bash
    sudo apt install ruby-dev ruby-bundler nodejs
    ```
    On MacOS the commands are:
    ```bash
    brew install ruby
    brew install node
    gem install bundler
    ```
1. Run `bundle install` to install ruby dependencies. If you get errors, delete Gemfile.lock and try again.
1. Run `jekyll serve -l -H localhost` to generate the HTML and serve it from `localhost:4000` the local server will automatically rebuild and refresh the pages on change.

If you are running on Linux it may be necessary to install some additional dependencies prior to being able to run locally: `sudo apt install build-essential gcc make`

## Using Docker

Working from a different OS, or just want to avoid installing dependencies? You can use the provided `Dockerfile` to build a container that will run the site for you if you have [Docker](https://www.docker.com/) installed.

Start by build the container:

```bash
docker build -t jekyll-site .
```

Next, run the container:
```bash
docker run -p 4000:4000 --rm -v $(pwd):/usr/src/app jekyll-site
```

To run the `docker run` command on Windows, you need to adjust the syntax for the volume mapping (`-v`) as Windows uses different path formats. Here's how to run your command on Windows:

### Steps for Windows:
1. **Check Docker Installation**: Ensure Docker is installed and running.
2. **Adjust Path for Volume Mapping**:

   - On Windows, replace `$(pwd)` with the full absolute path to your current directory. For example:

     ```bash
     -v C:\path\to\your\site:/usr/src/app
     ```

### Full Command Example:
```bash
docker run -p 4000:4000 --rm -v C:\path\to\your\site:/usr/src/app jekyll-site
```

### Things to Keep in Mind:
1. **Use PowerShell**:
   - If you are using PowerShell, you can use `${PWD}` for the current directory:
     ```bash
     docker run -p 4000:4000 --rm -v ${PWD}:/usr/src/app jekyll-site
     ```

2. **Enable Docker File Sharing**:
   - If your volume doesn't map correctly, ensure Docker has access to the drive where your project resides. To do this:
     - Open Docker Desktop.
     - Go to *Settings* → *Resources* → *File Sharing*.
     - Add your drive (e.g., `C:`).

3. **Run in Command Prompt or PowerShell**:
   - In *Command Prompt*:
   
     ```bash
     docker run -p 4000:4000 --rm -v C:\path\to\your\site:/usr/src/app jekyll-site
     ```
   - In *PowerShell*:

     ```bash
     docker run -p 4000:4000 --rm -v ${PWD}:/usr/src/app jekyll-site
     ```
