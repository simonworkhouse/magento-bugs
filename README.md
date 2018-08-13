# Magento 2 bug testing
Testing Magento 2 bugs

## Steps used to create this project
1. composer create-project --repository-url=https://repo.magento.com/ magento/project-community-edition ./magento/
2. ./bin/magento setup:install --backend-frontname=backend --db-host=? --db-name=? --db-user=? --db-password=? --base-url=? --admin-user=? --admin-password=? --admin-email=? --admin-firstname=? --admin-lastname=?
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
- Both categories and products can have the same url_rewrite request_path if the catalog/seo/product_url_suffix and/or catalog/seo/category_url_suffix changes
-- This can also occur if catalog/seo/product_use_categories changes to allow a product to match a category path 
- Categories with the same URL key will only allow the last category to have it's url_rewrites updated
- The same as above but for products
- If a category URL is updated and a child category has a conflict with a product in the same category, the URL rewrite generation will rollback the changes but still report a conflict which has a link to nothing.
- === Fixed === Stores are not imported from the config.php on install
- Category URL rewrites are not generated for additional stores even when they are created on install
- Cache key doesn't update properly for a different store id if the base URL is the same. Magento\Framework\App\PageCache\Identifier prefers the vary string.
- Importer: Failed categories aren't removed from the categories returned by upsertCategories
- No option for SVG support. The list of extensions are hard-coded into the relevant class.
- CMS block search doesn't do partial matches
- If a custom option is defined without values a fatal error will occur (this happens due to import data)
- If a configurable product has special prices start and end dates, these can't be updated in the admin interface as the fields aren't present. However; these fields are still sent across in the post data.
- Cannot save special price dates for en_AU locale.
- Exporting sample data and immediately attempting to import it fails.


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
3. ./bin/magento setup:install --backend-frontname=backend --db-host=? --db-name=? --db-user=? --db-password=? --base-url=? --admin-user=? --admin-password=? --admin-email=? --admin-firstname=? --admin-lastname=?
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