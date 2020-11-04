# SetupTripal GitHub Action

Provides a GitHub Action to setup a basic Tripal site in your GitHub Action Workflows. Specifically, you provide a PostgreSQL service and php in your workflow, then this action will download and install Drush 8.x, Drupal 7.x, Tripal 3.x and required Drupal extension modules.

## Input

The following input is required to connect to your existing PostgreSQL service. The values of these input should match directly to your service.

 - **postgres_user**: The user for your postgresql service.
 - **postgres_pass**: The password for your postgresql user.
 - **postgres_db**: The name of the database to install Tripal in.
 
 The following input are optional. They allow you to configure the administrative user of your Drupal/Tripal site.
 
 - **account_name**: The name of the administrative user for your Drupal/Tripal site.
 - **account_pass**: The password for your Drupal/Tripal administrative user.
 
 This final input allows you to specify the version of Tripal to use. The value should be the tag identifier of a release (e.g.  `7.x-3.4`) or the name of the development branch (i.e. 7.x-3.x) for the most recent development version.
 
 - **tripal_version**: The version of Tripal to download.

## Output

 - **drush_path**: The Path to the Drush Executable.
 - **drupal_root**: The Path to the Drupal installation.
 
## Usage

This action was developed for CI testing using PHPUnit for Tripal Core and extension modules. The following is an example github workflow for the `example_module` Tripal extension module. To adapt for your own usage, create a `.github/workflows/phpunit.yml` file with the following contents where `example_module` is the name of your module.

```yml
name: PHPUnit

# Controls when the workflow will run.
# Run this workflow every time a new commit is pushed to your repository
on: [push, pull_request]

jobs:
  run-tests:
    # The type of runner that the job will run on
    runs-on: ubuntu-latest
    # Matrix Build for this job.
    strategy:
      matrix:
        php-versions: ['7.1', '7.2']
    # Name the matrix build so we can tell them apart.
    name: PHPUnit Testing (PHP ${{ matrix.php-versions }})

    # Service containers to run with `run-tests`
    services:
      # Label used to access the service container
      postgres:
        # Docker Hub image
        image: postgres
        env:
          POSTGRES_USER: tripaladmin
          POSTGRES_PASSWORD: somesupersecurepassword
          POSTGRES_DB: testdb
        # Set health checks to wait until postgres has started
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          # Maps tcp port 5432 on service container to the host
          - 5432:5432

    steps:
    # Checkout the repository and setup workspace.
    - uses: actions/checkout@v2

    # Setup PHP according to the version passed in.
    - name: Setup PHP
      uses: shivammathur/setup-php@v2
      with:
        php-version: ${{ matrix.php-versions }}
        extensions: mbstring, intl, php-pgsql, php-gd, php-xml
        ini-values: memory_limit=2G
        coverage: xdebug
        tools: composer, phpunit
   
    # Install Drush/Drupal/Tripal
    - name: Setup Drush, Drupal 7.x, Tripal 3.x
      id: tripalsetup
      uses: tripal/setup-tripal-action@7.x-3.x-1.0
      with:
        postgres_user: tripaladmin
        postgres_pass: somesupersecurepassword
        postgres_db: testdb
  
    # Install Tripal Extension Module.
    - name: Install Tripal Extension Module
      id: installextension
      env:
        DRUSH: ${{ steps.tripalsetup.outputs.drush_path }}
        DRUPAL_ROOT: ${{ steps.tripalsetup.outputs.drupal_root }}
      run: |
        mkdir -p $DRUPAL_ROOT/sites/all/modules/example_module
        cp -R * $DRUPAL_ROOT/sites/all/modules/example_module
        cd $DRUPAL_ROOT
        $DRUSH en -y example_module
 
    # Runs the PHPUnit tests.
    # https://github.com/mheap/phpunit-github-actions-printer is used
    # to report PHPUnit fails in a meaningful way to github in PRs.
    - name: PHPUnit Tests
      env:
        DRUSH: ${{ steps.tripalsetup.outputs.drush_path }}
        DRUPAL_ROOT: ${{ steps.tripalsetup.outputs.drupal_root }}
      run: |
        cd $DRUPAL_ROOT/sites/all/modules/example_module
        composer require --dev mheap/phpunit-github-actions-printer --quiet
        composer update --quiet
        ./vendor/bin/phpunit --printer mheap\\GithubActionsReporter\\Printer
```
