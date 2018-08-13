# Magento 2 bug testing
Testing Magento 2 bugs

## Steps used to create this project
1. composer create-project --repository-url=https://repo.magento.com/ magento/project-community-edition ./magento/
2. ./bin/magento setup:install --backend-frontname=backend --db-host=? --db-name=? --db-user=? --db-password=? --base-url=? --admin-user=? --admin-password=? --admin-email=? --admin-firstname=? --admin-lastname=?
3. ./bin/magento deploy:mode:set developer
4. ./bin/magento sampledata:deploy
5. ./bin/magento setup:upgrade
6. ./bin/magento indexer:reindex
