# Magento 2 bug testing
Testing Magento 2 bugs

## Steps used to create this project
1. composer create-project --repository-url=https://repo.magento.com/ magento/project-community-edition ./magento/
2. ./bin/magento setup:install --backend-frontname=? --db-host=? --db-name=? --db-user=? --db-password=? --base-url=? --admin-user=? --admin-password=? --admin-email=? --admin-firstname=? --admin-lastname=?
3. ./bin/magento deploy:mode:set developer
4. ./bin/magento cache:disable
5. ./bin/magento sampledata:deploy
6. ./bin/magento setup:upgrade
7. ./bin/magento indexer:reindex

## Bug 17378
1. Run the project setup
2. Export all product catalog data
3. Change "catalog/seo/product_url_suffix" and "catalog/seo/category_url_suffix" to NULL.

Bugs: 
- === Reported === Both categories and products can have the same url_rewrite request_path if the catalog/seo/product_url_suffix and/or catalog/seo/category_url_suffix changes -- This can also occur if catalog/seo/product_use_categories changes to allow a product to match a category path 
- === Reported === Categories with the same URL key will only allow the last category to have it's url_rewrites updated
- === Reported === The same as above but for products
- === Reported === If a category URL is updated and a child category has a conflict with a product in the same category, the URL rewrite generation will rollback the changes but still report a conflict which has a link to nothing.
- === Fixed === Stores are not imported from the config.php on install
- === Reported === Category URL rewrites are not generated for additional stores even when they are created on install
- Cache key doesn't update properly for a different store id if the base URL is the same. Magento\Framework\App\PageCache\Identifier prefers the vary string.
- === Reported === Importer: Failed categories aren't removed from the categories returned by upsertCategories
- No option for SVG support. The list of extensions are hard-coded into the relevant class.
- === Reported === CMS block search doesn't do partial matches
- If a custom option is defined without values a fatal error will occur (this happens due to import data)
- If a configurable product has special prices start and end dates, these can't be updated in the admin interface as the fields aren't present. However; these fields are still sent across in the post data.
- Cannot save special price dates for en_AU locale.
- === Reported === Exporting sample data and immediately attempting to import it fails. -- Says: 1. Options for downloadable products not found in row(s): 46, 47, 48, 49, 50, 51
- === Reported === Typo in CMS storage class "keepRation"




## CMS block search doesn't do partial matches
When searching for blocks in the admin panel under "Content/Blocks" no results are displayed for partial matches for either the block title or identifier.
Although title will match for whole word matches.

### Preconditions
1. Magento 2.2.5
2. PHP 7.1.20
3. MySQL 5.7.23

### Steps to reproduce
1. composer create-project --repository-url=https://repo.magento.com/ magento/project-community-edition {install dir}
2. cd {install dir}
3. ./bin/magento setup:install --backend-frontname=? --db-host=? --db-name=? --db-user=? --db-password=? --base-url=? --admin-user=? --admin-password=? --admin-email=? --admin-firstname=? --admin-lastname=?
4. Create a new block with the title "Some test block" and identifier "some_test_identifier" under "Content/Blocks" in the admin panel.
5. Use the search input on the "Content/Blocks" index page and search for either "bloc" or "test_identifier"

### Expected Result
The test block that we created should be shown for both search queries

### Actual Result
No blocks can be found with those search queries.





## Magento\CatalogImportExport\Model\Import\Product\CategoryProcessor Failed categories aren't removed from the categories returned by upsertCategories
The function upsertCategories may return a category id that is no longer present in the DB as upsertCategory on line 138 of Magento\CatalogImportExport\Model\Import\Product\CategoryProcessor fetches the category id from the local class cache ($this->categories) of the available categories.
This causes the import to fail if the category was inserted, added to the cache and then removed from the DB due to either a transaction being rolled back or the category being automatically deleted.
Transactions in this instance are probably being rolled back due to url_key conflicts.
 
### Preconditions
1. Magento 2.2.5
2. PHP 7.1.20
3. MySQL 5.7.23

### Steps to reproduce
My best guess is that transactions are being rolled back due to url_key conflicts.
Refer to the issue https://github.com/magento/magento2/issues/17586 for clarification on how to set up import data that will have unexpected URL rewrite conflicts.
It would take quite a bit of time to set up a full procedure, but let me know if it's absolutely required.
It should be obvious enough for anyone reviewing the code and referring to the previous bug report https://github.com/magento/magento2/issues/17586

I have implemented a work-around for now, but this needs to be addressed and fixed in the core.

```
<?php

namespace {my_namespace}\Plugin\Magento\CatalogImportExport\Model\Import\Product;

use Magento\CatalogImportExport\Model\Import\Product\CategoryProcessor;

class CategoryProcessorPlugin
{
    /**
     * // NOTE: Must cache the failed categories here as Magento caches them even if they have failed and as 
     *          a result won't add them to the failed categories list
     * @var array
     */
    protected static $failedCategoriesCache = [];

    /**
     * [afterUpsertCategories description]
     * @param  CategoryProcessor $categoryProcessor [description]
     * @param  [type]            $categoryIds       [description]
     * @return [type]                               [description]
     */
    public function afterUpsertCategories(CategoryProcessor $categoryProcessor, $categoryIds)
    {
        // NOTE: Work-around for a Magento bug where it's still adding failed categories to the list
        if ($failedCategories = $categoryProcessor->getFailedCategories()) {
            foreach ($failedCategories as $failedCategory) {
                self::$failedCategoriesCache[] = $failedCategory['category']->getId();
            }
        }
        $categoryIds = array_diff($categoryIds, self::$failedCategoriesCache);
        return $categoryIds;
    }
}
```

### Expected Result
The function upsertCategories should only ever return categories that actually exist.

### Actual Result
The function upsertCategories returns categories from the local class cache that have since been removed from the DB during the same import run.
 




## Typo in Magento\Cms\Model\Wysiwyg\Images\Storage function resizeFile($source, $keepRation = true)
The parameter $keepRation should be called $keepRatio

### Preconditions
1. Magento 2.2.5

### Steps to reproduce
Look at the resizeFile function on line 572 of the Magento\Cms\Model\Wysiwyg\Images\Storage class.

### Expected Result
The parameter $keepRation should be $keepRatio

### Actual Result
The parameter is called $keepRation





## Category URL rewrites not generated after changes to "catalog/seo/category_url_suffix", "catalog/seo/product_url_suffix" or "catalog/seo/product_use_categories"
When updating the values of "catalog/seo/category_url_suffix", "catalog/seo/product_url_suffix" or "catalog/seo/product_use_categories" the entries in the url_rewrite table aren't generated except when a categories url_key is updated.
There doesn't appear to be any mechanism to re-generate these URL rewrites when these configuration values change. 

### Preconditions
1. Magento 2.2.5
2. PHP 7.1.20
3. MySQL 5.7.23

### Steps to reproduce
1. composer create-project --repository-url=https://repo.magento.com/ magento/project-community-edition {install dir}
2. cd {install dir}
3. ./bin/magento setup:install --backend-frontname=? --db-host=? --db-name=? --db-user=? --db-password=? --base-url=? --admin-user=? --admin-password=? --admin-email=? --admin-firstname=? --admin-lastname=?
4. ./bin/magento sampledata:deploy
5. ./bin/magento setup:upgrade
6. Update the values for "catalog/seo/product_url_suffix" and "catalog/seo/category_url_suffix" to ".php", the value for "catalog/seo/product_use_categories" to "1" and run "./bin/magento app:config:import"

### Expected Result
URL rewrites to be generated in the url_rewrite table and the URLs on the frontend should match the new suffixes and format.

### Actual Result
No URL rewrites are generated and the frontend still displays the old URL format and suffixes.





## URL rewrite generation has a number of significant issues
There are a number of conditions where the entries in the url_rewrite table can conflict sometimes silently failing to update the rewrites, sometimes showing links to unexpected conflicts and sometimes linking to conflicts that do not exist.

### Preconditions
1. Magento 2.2.5
2. PHP 7.1.20
3. MySQL 5.7.23

### Steps to reproduce
#### Initial setup
1. composer create-project --repository-url=https://repo.magento.com/ magento/project-community-edition {install dir}
2. cd {install dir}
3. ./bin/magento setup:install --backend-frontname=? --db-host=? --db-name=? --db-user=? --db-password=? --base-url=? --admin-user=? --admin-password=? --admin-email=? --admin-firstname=? --admin-lastname=?
4. ./bin/magento deploy:mode:set developer
5. ./bin/magento app:config:dump
6. ./bin/magento cache:disable

#### Scenario one
Updating the value of "catalog/seo/category_url_suffix" can cause Magento to silently fail to update the entries in the url_rewrite table for a specific category
1. Perform the steps described in "Initial setup"
2. Confirm that the value for "catalog/seo/category_url_suffix" is ".html" (the default value)
3. Proceed to the admin panel and create a sub-category called "Sale" with the URL key of "sale"
4. Create a sub-category under "Sale" called "Sandwiches" with the URL key of "sandwiches"
5. Now update the name of the new category from "Sandwiches" to "Sandwiches Old" (NOTE: This step is just to simulate user behaviour and highlight the potential of conflicts when importing data)
6. Update the value of "catalog/seo/category_url_suffix" to ".php" and run "./bin/magento app:config:import"
7. Create another sub-category under "Sale" called "Sandwiches" with the URL key of "sandwiches"
8. Update the value of "catalog/seo/category_url_suffix" to ".test" and run "./bin/magento app:config:import"
9. Update the URL key for the "Sale" category from "sale" to "sale-test"

#### Scenario two
Updating the values of "catalog/seo/product_url_suffix", "catalog/seo/category_url_suffix" and "catalog/seo/product_use_categories" can cause Magento to fail to update the entries in the url_rewrite table while showing an error with a link to a non-existent rewrite.
1. Perform the steps described in "Initial setup"
2. Confirm that the values for "catalog/seo/product_url_suffix" and "catalog/seo/category_url_suffix" are ".html" and the value for "catalog/seo/product_use_categories" is "0" (the default values)
3. Proceed to the admin panel and create a sub-category called "Sale" with the URL key of "sale"
4. Create a sub-category under "Sale" called "Sandwiches" with the URL key of "sandwiches"
5. Update the value of "catalog/seo/product_url_suffix" and "catalog/seo/category_url_suffix" to ".php" and run "./bin/magento app:config:import"
6. Create a product called "Sandwiches" with the URL key of "sandwiches" and add it to the "Sale" category
7. Update the values for "catalog/seo/product_url_suffix" and "catalog/seo/category_url_suffix" to ".test", the value for "catalog/seo/product_use_categories" to "1" and run "./bin/magento app:config:import"
8. Update the URL key for the "Sale" category from "sale" to "sale-test"

#### Scenario three
Scenario three is essentially combinations of the procedures in scenarios one and two except where a valid link to the matching url_rewrite entry is displayed.
This scenario causes a number of issues with the import process but that will have to be reported as a separate bug. 

### Expected Result
#### Scenario one
The URL keys for categories should never have been allowed to conflict like this and at the very least a warning notifying the admin user of conflicts in the url_rewrite table should be shown.

#### Scenario two
Same as scenario one except that there should be a much better way of managing URL rewrite conflicts.

#### Scenario three
Essentially the same as scenario one

### Actual Result 
#### Scenario one 
A success message saying "You saved the category." is displayed to the admin user.

#### Scenario two
An error stating tho following is displayed:
```
The value specified in the URL Key field would generate a URL that already exists.
To resolve this conflict, you can either change the value of the URL Key field (located in the Search Engine Optimization section) to a unique value, or change the Request Path fields in all locations listed below:
   
- sale-test/sandwiches.test
```
The link "sale-test/sandwiches.test" points to a record in the url_rewrite table that doesn't exist as the transaction that contained the insert for this entry was rolled back.

#### Scenario three
Unexpected URL conflicts are shown



## Exporting product data generates a CSV that doesn't pass the import check process
From an clean install with sample data deployed, if you export product data via "System/Data Transfer/Export" then immdediately try to import it using "System/Data Transfer/Import" it fails to pass the initial data validation checks.
The error received is:
1. Options for downloadable products not found in row(s): 46, 47, 48, 49, 50, 51

### Preconditions
1. Magento 2.2.5
2. PHP 7.1.20
3. MySQL 5.7.23

### Steps to reproduce
1. composer create-project --repository-url=https://repo.magento.com/ magento/project-community-edition {install dir}
2. cd {install dir}
3. ./bin/magento setup:install --backend-frontname=? --db-host=? --db-name=? --db-user=? --db-password=? --base-url=? --admin-user=? --admin-password=? --admin-email=? --admin-firstname=? --admin-lastname=?
4. ./bin/magento sampledata:deploy
5. ./bin/magento setup:upgrade
6. Visit the admin panel run an export and then an import of that exported data.

#### Export settings
- Entity Type: Products
- Export File Format: CSV (default)
- Fields Enclosure: Not checked (default)
- No entity attributes excluded (default)

#### Import settings
- Entity Type: Products
- Import Behavior: Add/Update
- Stop on Error (default)
- Allowed Errors Count: 10 (default)
- Field separator: , (default)
- Multiple value separator: , (default)
- Fields Enclosure: Not checked (default)

### Expected Result
Import to be successful

### Actual Result
Import fails with the error:
1. Options for downloadable products not found in row(s): 46, 47, 48, 49, 50, 51




## Category URL rewrites not generated for new stores
When a new website/store is created URL rewrites are only generated for the new store if a category's url_key is updated.
There doesn't appear to be any mechanism to re-generate these URL rewrites for a store, and they also don't take into account any changes to the "catalog/seo/category_url_suffix" configuration value. 

### Preconditions
1. Magento 2.2.5
2. PHP 7.1.20
3. MySQL 5.7.23

### Steps to reproduce
1. composer create-project --repository-url=https://repo.magento.com/ magento/project-community-edition <install dir>
2. cd <install dir>
3. ./bin/magento setup:install --backend-frontname=? --db-host=? --db-name=? --db-user=? --db-password=? --base-url=? --admin-user=? --admin-password=? --admin-email=? --admin-firstname=? --admin-lastname=?
4. ./bin/magento sampledata:deploy
5. ./bin/magento setup:upgrade
6. Visit the admin panel and create a new website, store and store view using the same root category as the default store.

### Expected Result
Visiting category URLs on the frontend of the new store should match the URLs on the frontend of the default store.
Also, entries for the now store should exist in the url_rewrite table.

### Actual Result
URLs for the new store are in the format "/catalog/category/view/s/{url_key}/id/{category_id}/" while the default store has the expected URL format.
Also, no entries for the new store exist in the url_rewrite table and there doesn't appear to be any way to generate these short of updating the URL key for each category.  




## "./bin/magento config:show" fails
The ./bin/magento config:show command fails with a fatal error after running ./bin/magento app:config:dump
Similar issue reported previously https://github.com/magento/magento2/issues/16654

### Preconditions
1. Magento 2.2.5
2. PHP 7.1.20
3. MySQL 5.7.23

### Steps to reproduce
1. composer create-project --repository-url=https://repo.magento.com/ magento/project-community-edition <install dir>
2. cd <install dir>
3. ./bin/magento setup:install --backend-frontname=? --db-host=? --db-name=? --db-user=? --db-password=? --base-url=? --admin-user=? --admin-password=? --admin-email=? --admin-firstname=? --admin-lastname=?
4. ./bin/magento app:config:dump
5. ./bin/magento config:show

### Expected Result
The output of all config values

### Actual Result
...
catalog/productalert_cron/error_email - 
catalog/product_video/play_if_base - 0
catalog/product_video/show_related - 0
catalog/product_video/video_auto_restart - 0
catalog/review/allow_guest - 1
catalog/search/engine - mysql
catalog/search/min_query_length - 1
catalog/search/max_query_length - 128
catalog/search/max_count_cacheable_search_terms - 100
analytics/subscription/enabled - 1
PHP Fatal error:  Uncaught Error: Call to undefined method Magento\Config\Model\Config\Structure\Element\Group\Interceptor::hasBackendModel() in <install dir>/vendor/magento/module-config/Console/Command/ConfigShow/ValueProcessor.php:100
Stack trace:
\#0 <install dir>/vendor/magento/module-config/Console/Command/ConfigShowCommand.php(197): Magento\Config\Console\Command\ConfigShow\ValueProcessor->process('default', '', 'Magento Analyti...', 'analytics/integ...')
\#1 <install dir>/vendor/magento/module-config/Console/Command/ConfigShowCommand.php(202): Magento\Config\Console\Command\ConfigShowCommand->outputResult(Object(Symfony\Component\Console\Output\ConsoleOutput), 'Magento Analyti...', 'analytics/integ...')
\#2 <install dir>/vendor/magento/module-config/Console/Command/ConfigShowCommand.php(202): Magento\Config\Console\Command\ConfigShow in <install dir>/vendor/magento/module-config/Console/Command/ConfigShow/ValueProcessor.php on line 100