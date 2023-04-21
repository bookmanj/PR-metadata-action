# PR-metadata-action
to create a javascript action
~~~
npm init -y
npm install @actions/core @actions/github
~~~

Create action.yml, and then write you action code in index.js

example action.yml
~~~
name: 'PR Metadata Action'
description: 'Adds pull request file changes as a comment to a newly opened PR'
inputs:
  owner:
    description: 'The owner of the repository'
    required: true
  repo:
    description: 'The name of the repository'
    required: true
  pr_number:
    description: 'The number of the pull request'
    required: true
  token:
    description: 'The token to use to access the GitHub API'
    required: true
runs:
  using: 'node16'
  main: 'index.js'
~~~

now create the index.js file and write your action code
~~~
const core = require('@actions/core');
const github = require('@actions/github');

const main = async () => {
  try {
    /**
     * We need to fetch all the inputs that were provided to our action
     * and store them in variables for us to use.
     **/
    const owner = core.getInput('owner', { required: true });
    const repo = core.getInput('repo', { required: true });
    const pr_number = core.getInput('pr_number', { required: true });
    const token = core.getInput('token', { required: true });

    /**
     * Now we need to create an instance of Octokit which will use to call
     * GitHub's REST API endpoints.
     * We will pass the token as an argument to the constructor. This token
     * will be used to authenticate our requests.
     * You can find all the information about how to use Octokit here:
     * https://octokit.github.io/rest.js/v18
     **/
    const octokit = new github.getOctokit(token);

    /**
     * We need to fetch the list of files that were changes in the Pull Request
     * and store them in a variable.
     * We use octokit.paginate() to automatically loop over all the pages of the
     * results.
     * Reference: https://octokit.github.io/rest.js/v18#pulls-list-files
     */
    const { data: changedFiles } = await octokit.rest.pulls.listFiles({
      owner,
      repo,
      pull_number: pr_number,
    });


    /**
     * Contains the sum of all the additions, deletions, and changes
     * in all the files in the Pull Request.
     **/
    let diffData = {
      additions: 0,
      deletions: 0,
      changes: 0
    };

    // Reference for how to use Array.reduce():
    // https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/Reduce
    diffData = changedFiles.reduce((acc, file) => {
      acc.additions += file.additions;
      acc.deletions += file.deletions;
      acc.changes += file.changes;
      return acc;
    }, diffData);

    /**
     * Loop over all the files changed in the PR and add labels according 
     * to files types.
     **/
    for (const file of changedFiles) {
      /**
       * Add labels according to file types.
       */
      const fileExtension = file.filename.split('.').pop();
      switch(fileExtension) {
        case 'md':
          await octokit.rest.issues.addLabels({
            owner,
            repo,
            issue_number: pr_number,
            labels: ['markdown'],
          });
        case 'js':
          await octokit.rest.issues.addLabels({
            owner,
            repo,
            issue_number: pr_number,
            labels: ['javascript'],
          });
        case 'yml':
          await octokit.rest.issues.addLabels({
            owner,
            repo,
            issue_number: pr_number,
            labels: ['yaml'],
          });
        case 'yaml':
          await octokit.rest.issues.addLabels({
            owner,
            repo,
            issue_number: pr_number,
            labels: ['yaml'],
          });
      }
    }

    /**
     * Create a comment on the PR with the information we compiled from the
     * list of changed files.
     */
    await octokit.rest.issues.createComment({
      owner,
      repo,
      issue_number: pr_number,
      body: `
        Pull Request #${pr_number} has been updated with: \n
        - ${diffData.changes} changes \n
        - ${diffData.additions} additions \n
        - ${diffData.deletions} deletions \n
      `
    });

  } catch (error) {
    core.setFailed(error.message);
  }
}

// Call the main function to run the action
main();
~~~


Next add ncc package so we can compile our code into a single file
~~~
npm i -g @vercel/ncc
~~~

Compile your code
~~~
ncc build index.js -o dist
~~~

This will create a new directory called dist and create a file called index.js that contains our code and all the packages we need to run our action.

Now we need to make sure our action.yml file contains the correct runs section. You need to replace:

~~~
runs:
  using: 'node16'
  main: 'index.js'
~~~

with:

~~~
runs:
  using: 'node16'
  main: 'dist/index.js'
~~~

create workflow file to call the action
~~~
mkdir -p .github/workflows
touch .github/workflows/main.yml
~~~

add the following to the main.yml file
~~~
name: PR metadata annotation

on: 
  pull_request:
    types: [opened, reopened, synchronize]

jobs:

  annotate-pr:
    runs-on: ubuntu-latest
    name: Annotates pull request with metadata
    steps:
      - name: Annotate PR
        uses: link-/PR-metadata-action@main
        with:
          owner: ${{ github.repository_owner }}
          repo: ${{ github.event.repository.name }}
          pr_number: ${{ github.event.number }}
          token: ${{ secrets.GITHUB_TOKEN }}
~~~

commit and push your changes

lastly, go to your repo and create a new pull request and you should see the action run and add labels to your PR