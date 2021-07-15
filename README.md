# Set up Astra using Terraform
## Set up Terraform on Mac OSX
```sh
brew tap hashicorp/tap
brew install hashicorp/tap/terraform
brew update
brew upgrade hashicorp/tap/terraform
terraform -help
```
## Export your secure token so Terraform can use it
This prevents you from having to set this information in the file.
```sh
export ASTRA_API_TOKEN=<your-token>
```

## Configure the Astra provider for Terraform
Create a file called `main.tf` and paste the following:
```js
// Define the Astra provider for Terraform
terraform {
  required_providers {
    astra = {
      source = "datastax/astra"
      version = "0.0.5-pre"
    }
  }
}

// Provide your Astra token
provider "astra" {
    // Will read from environment variable ASTRA_API_TOKEN, else define as follows:
    // token =
}
```
Then run `terraform init`.

## List your databases
Add the following:
```js
// Get your existing databases
data "astra_databases" "databaselist" {
  status = "ACTIVE"
}

// Output your existing databases
output "existing_dbs" {
  value = [for db in data.astra_databases.databaselist.results : "${db.id}, ${db.name}"]
}
```
Now run `terraform plan`.

## Get DB configuration from Astra to Terraform
Add the following to create a placeholder for the config:
```js
resource "astra_database" "example" {
  # (resource arguments)
}
```
Now run `terraform import astra_database.example <one of the DB id's you just retrieved>`.
You just created a Terraform state file defining your database!

## Now let's create a new database
Start out fresh and delete both `main.tf` and the `*.tfstate` file!
### List the potential regions
Create the following `main.tf` file:
```js
// Define the Astra provider for Terraform
terraform {
  required_providers {
    astra = {
      source = "datastax/astra"
      version = "0.0.5-pre"
    }
  }
}

// Provide your Astra token
provider "astra" {
    // Will read from environment variable ASTRA_API_TOKEN, else define as follows:
    // token =
}

// Get the regions that are available within Astra
data "astra_available_regions" "regions" {
}

// Output the regions
output "available_regions" {
  value = [for region in data.astra_available_regions.regions.results : "${region.cloud_provider}, ${region.region}, ${region.tier}" if region.tier == "serverless"]
}
```
Now run `terraform plan` and note the cloud providers and their regions.
### Create a new database
Pick a cloud provider and region from the previous list and create the following `main.tf`:
```js
// Define the Astra provider for Terraform
terraform {
  required_providers {
    astra = {
      source = "datastax/astra"
      version = "0.0.5-pre"
    }
  }
}

// Provide your Astra token
provider "astra" {
    // Will read from environment variable ASTRA_API_TOKEN, else define as follows:
    // token =
}

// Create the database and initial keyspace
resource "astra_database" "dev" {
  name           = "starship_enterprise"
  keyspace       = "life_support_systems"
  cloud_provider = "AWS"
  region         = "eu-central-1"
}

// Get the location of the secure connect bundle
data "astra_secure_connect_bundle_url" "dev" {
  database_id = astra_database.dev.id
}

// Output the created database id
output "database_id" {
  value = astra_database.dev.id
}

// Output the download location for the secure connect bundle
output "secure_connect_bundle_url" {
  value = data.astra_secure_connect_bundle_url.dev.url
}
```
Now run `terraform plan` to see what actions are going to be taken. When you're happy, run `terraform plan` and type `yes`.

Terraform will now create the Astra database and upon completion provide:
- The database id
- The download location for the secure connect bundle. It will be available for download for 5 minutes.

### Download the secure connect bundle
Copy the URL from above and paste it here:
```sh
wget -O secure_connect_bundle.zip "<url>"
```