# Automating Azure SQL Database Backups with Linux: A Step-by-Step Guide

In this post, I’ll walk you through the process of automating SQL database backups in Azure, using Linux bash scripting. This includes connecting to an Azure SQL Database, exporting the database to a .bacpac file, and uploading it to an Azure Storage account for safe keeping. 

You need to install the az cli and sqlpackage to run this script

## Create an Azure SQL Database
#### Login to Azure CLI:
```
az login
```

#### Create a Resource Group:
```
az group create --name MyResourceGroup --location eastus
```

#### Create an Azure SQL Server:
```
az sql server create \
    --name my-sql-server \
    --resource-group MyResourceGroup \
    --location eastus \
    --admin-user sqladmin \
    --admin-password MyP@ssw0rd123
```

#### Create an Azure SQL Database:
```
az sql db create \
    --resource-group MyResourceGroup \
    --server my-sql-server \
    --name MyDatabase \
    --service-objective S1
```

## Create an Azure Storage Account
#### Create the Storage Account:
```
az storage account create \
    --name mystorageaccount \
    --resource-group MyResourceGroup \
    --location eastus \
    --sku Standard_LRS
```

#### Create a Blob Container:
```
az storage container create \
    --account-name mystorageaccount \
    --name sqlbackups
```

## Create an Azure Key Vault
#### Create the Key Vault:
```
az keyvault create \
    --name MyKeyVault \
    --resource-group MyResourceGroup \
    --location eastus
```

#### Add Secrets to the Key Vault:
Store the SQL server credentials:
```
az keyvault secret set --vault-name MyKeyVault --name "sql-admin-username" --value "sqladmin"
az keyvault secret set --vault-name MyKeyVault --name "sql-admin-password" --value "MyP@ssw0rd123"
```

Store the Storage Account Key:
```
STORAGE_KEY=$(az storage account keys list --account-name mystorageaccount --query [0].value -o tsv)
az keyvault secret set --vault-name MyKeyVault --name "storage-account-key" --value "$STORAGE_KEY"
```

## Write the Backup Script

#### Set Up Variables: 
Define the Key Vault, SQL server, database name, storage account, and other parameters.
```
KEYVAULT="MyKeyVault"
DB_SERVER="my-sql-server.database.windows.net"
DB_NAME="MyDatabase"
STORAGE_ACCOUNT="mystorageaccount"
CONTAINER="sqlbackups"
BCKP_FILE="/tmp/backup_$(date +%F).bacpac"
```


#### Retrieve Credentials from Key Vault: 
Use Azure CLI to securely fetch the SQL DB credentials and storage account keys from Azure Key Vault.
```
DB_USER=$(az keyvault secret show --name "sqlusr" --vault-name "$KEYVAULT" --query value -o tsv)
DB_PASS=$(az keyvault secret show --name "sqlpw" --vault-name "$KEYVAULT" --query value -o tsv)
STORAGE_KEY=$(az keyvault secret show --name "storagekey" --vault-name "$KEYVAULT" --query value -o tsv)
```


#### Export SQL Database: 
Use sqlpackage to export the database to a .bacpac file.
```
sqlpackage /a:Export \
/scs:"Server=tcp:$DB_SERVER,1433;Initial Catalog=$DB_NAME;User_ID=$DB_USER;Password=$DB_PASS;" \
/tf:"$BCKP_FILE"
```


#### Upload to Azure Storage: 
Upload the .bacpac file to an Azure Blob Storage container.
```
az storage blob upload --acount-name "$STORAGE_ACCOUNT" --account-key "$STORAGE_KEY" --container-name "$CONTAINER" \
--file "$BCKP_FILE" --name "$(basename $BCKP_FILE)"
```


#### Remove Local File: 
Clean up by removing the backup file from the local server after upload.
```
rm -f "$BCKP_FILE"
```
