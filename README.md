# software-documentation

This repository contains a [MyST-based](https://mystmd.org) website for JupyerHealth software documentation. You can see a preview of it at [https://jupyterhealth.github.io/software-documentation](https://jupyterhealth.github.io/software-documentation).

## Preview Changes (Optional)

If you want to preview the website locally on your own computer before they go live, follow these instructions. It is not strictly necessary, but we recommend doing so to spot errors. If you are confident that your changes will not break anything (for example for quick fixes), you can skip this section.

1. Install a Conda distribution, such as [Miniforge](https://github.com/conda-forge/miniforge) or [Anaconda](https://www.anaconda.com/download#downloads), if you don't already have one installed.


2. Install software using conda/mamba. To preview the website on your own device you will need some command-line programs such as `nodejs` and [myst](https://mystmd.org/guide/quickstart). We have provided an `environment.yml` file to make this step simpler. Run the commands below to install all dependencies in a separate environment, and then activate it. This keeps your website development separate from any other projects you're working on.
     ```bash
     # You can replace `mamba` with `conda`.
     mamba env create -f environment.yml
     
     # Now you can activate the environment.
     # You can use `mamba activate` or `conda activate`,
     # but these require that you have run `mamba init` or `conda init`,
     # which modifies your shell.
     mamba activate jh-software
     
     # Alternatively, you can activate like this, without
     # having do the `init` step mentioned above.
     source activate jh-software
     ```

   Whenever you need to start up a new terminal to work on your website, first activate the the environment that contains the necessary programs with `mamba activate jh-software`. Otherwise your terminal may report that it cannot find `myst` or its dependencies.


3. Preview your changes on your own device. Run `myst init` in the working directory to initialize your working directory. When prompted, let the command run `myst start` for you to generate a preview. `myst` will display a localhost URL where the site preview is running, such as http://localhost:3000. Open this URL in your browser.

   **Note:** When `myst` detects problems in the markdown that prevent the website from rendering correctly, it will print the file and line number containing the issue. If this happens, fix the file and rerun `myst init`.

   **Note:** If you are on a Mac, `myst` may prompt you to allow `node` to accept incoming network connection. Allow this to enable website previewing to work.

   You can leave `myst` running as you make changes to the source files; saving changes to the source files will generally will be reflected live in your browser, though you may sometimes need to navigate away and back or reload the page.

   You can also run `myst build` to create the static HTML (in the `_build` directory) without automatically displaying it.

   **Note:** Do not commit the files in `_build` to your repository. They will not be used when the website is deployed. Rather, an automated process via GitHub Actions will run and create the static build files.


### Push Your Changes to Publish

1. Push your changed to GitHub:
   ```bash
   git push
   ```
   This will cause GitHub to rebuild your site.

2.  You can observe the build process at GitHub by clicking on the Actions button at the top of your repository, e.g. https://github.com/jupyterhealth/software-documentation/actions. It usually takes a couple of minutes for this to complete.

3. If there are no problems, your website will be publicly available at https://jupyterhealth.github.io/software-documentation.
