## Apply All Available Upstream Updates

[Update the site](/core-updates) to the latest [Pantheon Drops 8](https://github.com/pantheon-systems/drops-8) Upstream and apply all available updates.

1. Use Terminus to list all available updates:

  ```bash{outputLines:2}
  terminus upstream:updates:list $SITE
  [warning] There are no available updates for this site.
  ```

1. If any updates are available, apply them using the command line or via the [Pantheon Dashboard](/core-updates#apply-upstream-updates-via-the-site-dashboard):

  ```bash{promptUser: user}
  terminus upstream:updates:apply $SITE.dev --updatedb
  ```

## Add the Pantheon Integrated Composer Upstream in a New Local Branch

This process involves significant changes to the codebase. We recommend you to do this work on a new branch, as it might take you some time to complete and rolling back changes can be complicated:

1. In your local terminal, change directories to the site project. For example, if you keep your projects in a folder called `projects` in the home directory:

  ```bash{promptUser:user}
  cd ~/projects/$SITE/
  ```

1. Add the Pantheon Drupal Upstream as a new remote called `ic`, fetch the `ic` branch, and checkout to a new local branch based on it called `composerify`:

  ```bash{promptUser:user}
  git remote add ic git@github.com:pantheon-upstreams/drupal-project.git && git fetch ic && git checkout -b composerify ic/master
  ```

  If you prefer, you can replace `composerify` with another branch name. If you do, remember to adjust the other examples in this doc to match.

1. Copy any existing configuration from the default branch. If no files are copied through this step, that's ok:

  ```bash{promptUser:user}
  git checkout master sites/default/config
  git mv sites/default/config/* config
  git rm -f sites/default/config/.htaccess
  git commit -m "Pull in configuration from default branch"
  ```

1. Check for `pantheon.yml` settings you need to preserve by comparing your old codebase's `pantheon.yml` to the new `pantheon.upstream.yml`:

  ```bash{promptUser:user}
  git diff master:pantheon.yml pantheon.upstream.yml
  ```

  Press `q` on your keyboard to exit the diff display.

   - If there are settings from `pantheon.yml` (shown with a `-` in the diff output), consider copying over your old `pantheon.yml` to preserve these settings:

     ```bash{promptUser:user}
     git checkout master pantheon.yml
     git add pantheon.yml
     git commit -m 'Copy my pantheon.yml'
     ```

   If you prefer to keep the value for `database` from `pantheon.upstream.yml`, remove it from `pantheon.yml`.

## Add in the Custom and Contrib Code Needed to Run Your Site

What makes your site code unique is your selection of contributed modules and themes, and any custom modules or themes your development team has created. These customizations need to be replicated in your new project structure.

### Contributed Code

#### Modules and Themes

The goal of this process is to have Composer manage all the site's contrib modules, contrib themes, core upgrades, and libraries (we'll call this _contributed code_). The only things that should be migrated from the existing site are custom code, custom themes, and custom modules that are specific to the existing site.

The steps here ensure that any modules and themes from [drupal.org](https://drupal.org) are in the `composer.json` `require` list.

Once Composer is aware of all the contributed code, you'll be able to run `composer upgrade` from within the directory to have Composer upgrade all the contributed code automatically.

Begin by reviewing the existing site's code. Check for contributed modules in `/modules`, `/modules/contrib`, `/sites/all/modules`, and `/sites/all/modules/contrib`.

1. When reviewing the site, take stock of exactly what versions of modules and themes you depend on. One way to do this is to change to run the `pm:projectinfo` Drush command from within a contributed modules folder (e.g. `/modules`, `/themes`, `/themes/contrib`, `/sites/all/themes`, `/sites/all/themes/contrib`, etc.).

  ```bash{promptUser:user}
  terminus drush $SITE.dev -- pm:projectinfo --fields=name,version --format=table
  ```

  This will list each module followed by the version of that module that is installed.

1. You can add these modules to your new codebase using Composer by running the following for each module in the `$SITE-composer` directory:

  ```bash{promptUser:user}
  composer require drupal/MODULE_NAME:^VERSION
  ```

  Where `MODULE_NAME` is the machine name of the module in question, and `VERSION` is the version of that module the site is currently using. Composer may pull in a newer version than what you specify, depending upon what versions are available. You can read more about the caret (`^`) in the [Composer documentation](https://getcomposer.org/doc/articles/versions.md#caret-version-range-).

  Some modules use different version formats.

   - For older-style Drupal version strings:

   ```none
   Chaos Tools (ctools)  8.x-3.4
   ```

    Replace the `8.x-` to convert this into `^3.4`

   - Semantic Versioning version strings:

   ```none
   Devel (devel)  4.1.1
   ```

    Use the version directly, e.g. `^4.1.1`

  If you get the following error, the module listed in the error (or its dependencies) does not meet compatibility requirements:

   ```none
   [InvalidArgumentException]
   Could not find a version of package drupal/MODULE_NAME matching your minimum-stability (stable). Require it with an explicit version constraint allowing its desired stability.
   ```

   If there is no stable version you can switch to, you may need to adjust the `minimum-stability` setting of `composer.json` to a more relaxed value, such as `beta`, `alpha`, or `dev` (not recommended). You can read more about `minimum-stability` in the [Composer documentation](https://getcomposer.org/doc/04-schema.md#minimum-stability).

     - If a dev version of a module fails because it requires a dev version of a dependency, allowlist the dev dependency in the same `composer require` as the module:

     ```bash{promptUser:user}
     composer require drupal/some-module:^1@dev org/some-dependency:^2@dev
     ```

<!-- commenting out until the script has a proper place to live

**Trust a robot?**

One of Pantheon's engineers got tired of doing this process by hand, so he trained some robots to identify modules and put them into `composer.json`.

Robots are cool, but they're not perfect, so you should understand the goal of this process as well as the limitations of automating the process.

<Accordion title="A script that can help add modules to composer.json" id="modules-script" icon="cogs">

First, disclaimers:

This script is provided without warranty or direct support. Issues and questions may be filed in GitHub but their resolution is not guaranteed.

Proceed at your own risk. Automation is better when you understand what it's doing.

- The script does not resolve `composer.json` version problems.

  - If there's a version conflict when you run `composer install`, you will need to resolve the conflicts yourself.

- The script does not check for Drupal 9 compatibility.

  - Use the [Upgrade Status](https://drupal.org/project/upgrade_status) module to check for Drupal 9 compatibility.

- The script does not resolve module stability.

  - If you get the following error, the module listed in the error (or its dependencies) does not meet compatibility requirements:

   ```none
   [InvalidArgumentException]
   Could not find a version of package drupal/MODULE_NAME matching your minimum-stability (stable). Require it with an explicit version constraint allowing its desired stability.
   ```

   If there is no stable version you can switch to, you may need to adjust the `minimum-stability` setting of `composer.json` to a more relaxed value, such as `beta`, `alpha`, or even `dev`. You can read more about `minimum-stability` in the [Composer documentation](https://getcomposer.org/doc/04-schema.md#minimum-stability).

- The script does not resolve patches.

  Many times a module will be "patched" or have a `.patch` file that fixes known issues before the fix is available in the downloaded version. This script does not attempt to resolve any patches.

If you still want to try it:

1. Say the following out loud: "Pantheon is not responsible for what I am about to do."

1. Create a directory called `bin` within your repository and copy the `migrateModules.php` file from [GitHub](https://github.com/pantheon-upstreams/drupal-project/pull/8/files#diff-c68470275758ca395b98d53ed258b63519435d492dd51531c4cf372814c6593e) into that directory.

1. To run the module migration script, `cd` to the root directory of the repository and run `bin/migrateModules.php`.

1. To see which modules were added, run `git diff composer.json`.

1. Remove the `composer.lock` file. Composer will create a new one with the modules you just added.

1. Run `composer install` and resolve any remaining version conflicts.

</Accordion>
-->

#### Libraries

Libraries can be handled similarly to modules, but the specifics depend on how your library code was included in the source site. If you're using a library's API, you may have to do additional work to ensure that library functions properly.

### Custom Code

Manually copy custom code from the existing site repository to the Composer-managed directory.

#### Modules and Themes

Modules:

```bash{promptUser:user}
mkdir -p ../$SITE-composer/web/modules/custom # create the directory if it doesn't already exist
cp -r modules/custom/awesome_module ../$SITE-composer/web/modules/custom
```

Themes:

```bash{promptUser:user}
mkdir -p ../$SITE-composer/web/themes/custom # create the directory if it doesn't already exist
cp -r themes/custom/great_theme ../$SITE-composer/web/themes/custom
```

Follow suit with any other custom code you need to carry over.

#### Settings.php

Your existing site may have customizations to `settings.php` or other configuration files. Review these carefully and extract relevant changes from these files to copy over. Always review any file paths referenced in the code, as these paths may change in the transition to Composer.

We don't recommend that you completely overwrite the `settings.php` file with the old one, as it contains customizations for moving the configuration directory you don't want to overwrite, as well as platform-specific customizations.

```bash{promptUser:user}
git status # Ensure working tree is clean
git show master:sites/default/settings.php > web/sites/default/original-settings.php
diff -Nup --ignore-all-space web/sites/default/settings.php web/sites/default/original-settings.php
# edit web/sites/default/settings.php and commit as needed
rm web/sites/default/original-settings.php
```

The resulting `settings.php` should have no `$databases` array.

#### Configuration

If you are using an exported config, you will need to move the configuration files to a new location. The preferred (and assumed) location of the configuration directories when using a nested docroot and Composer is at the root of the repository next to the web directory:

```none
 site-composer
|-web
|-config    <--Here!
|-vendor
|-composer.json
|-etc...
```

Locate the configuration files in the existing site and move them here. If they are stored in the files directory on your existing site, retrieve them via [SFTP](/sftp), as the Git clone would not contain them. The example project is configured to use this location.

## Deploy

You've now committed the code to the local branch. Deploy that branch directly to a new Multidev and test the site in the browser. If the site doesn't load properly, clear the cache. If there are any issues, utilize the site's logs via `terminus drush $SITE.composerify -- wd-show` to inspect the watchdog logs, or follow the directions in our documentation on [log collection](/logs).

### Deploy to a Multidev

Push the changes to a Multidev called `composerify` to safely test the site without affecting the Dev environment:

```bash{promptUser:user}
git add .
git commit -m "convert to composer"
git push origin composerify && terminus env:create $SITE.dev composerify
```

Once you have confirmed the site is working, merge `composerify` into `master`, and follow the standard [relaunch workflow](/relaunch) to QA a code change before going live.
