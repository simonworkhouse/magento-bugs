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
3. Change "catalog.seo.product_url_suffix" and "catalog.seo.category_url_suffix" to NULL.
