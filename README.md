# Restoring seventh-legion.net

The purpose of this document is to describe the restoration of the seventh-legion MediaWiki deployment from an existing one.

## Credentials for the MediaWiki deployment

Assuming the EC2 instance has been deployed according to the instructions in this guide, the credentials for logging into SQL can be found by running the command sudo cat `/home/bitnami/bitnami_credentials`.

## Backing up old instance

### Backing up the SQL database

If a backup is being performed from an existing deployment, connect to the EC2 instance and dump the existing SQL database:

```
mysqldump -u root -p bitnami_mediawiki > backup.sql
```

Download this file locally or transfer it directly to the new EC2 instance.

### Backing up the images

Create a compressed file of the `images` folder, which is the other component required to restore a MediaWiki instance. 

```
sudo tar zcvhf images.tgz /opt/bitnami/mediawiki/images
```
Download this file locally or transfer it directly to the new EC2 instance.

## Starting a new EC2 instance

To complete this step, you'll need an Amazon Web Services account. With that in hand, navigate the [AMI Catalog](https://us-east-2.console.aws.amazon.com/ec2/home?region=us-east-2#AMICatalog:) (example link is for the `us-east-2` region). Once there, navigate to the _AWS Marketplace AMIs_ tab and search for "bitnami mediawiki". There will likely be only one result, which should be chosen with the "Select" button to the right of it. Choose "Subscribe Now", and when the pop-up disappears, choose "Launch Instance with AMI".

When setting up the instance, one does need to choose a properly configured security group. That step is omitted from this guide, hopefully temporarily. One could either create or use an existing key pair.

## Installing the wiki on the new instance

These steps begin by connecting to the new EC2 instance using the .pem file chosen in the "Starting a new EC2 instance" step (ex. `ssh -i my_key.pem bitnami@<public_ip_of_instace>`).

### Restoring the SQL database

1. Get the credentials for the SQL database on the new instance (as above), then drop into it (`mysql -u root -p`). 
2. Drop the existing `bitnami_mediawiki` table: `DROP TABLE bitnami_mediawiki`.
3. Create a new (empty) table with the same name: `CREATE TABLE bitnami_mediawiki`.
4. Exit SQL and from the command line, load the SQL file generated above: `mysql -u root -p bitnami_mediawiki < backup.sql`.

### Restoring the images

1. Remove the existing images folder: `sudo rm -rf /opt/bitnami/mediawiki/images`.
2. Unpack the images.tgz file from the backup: `tar -xvzf images.tgz`
3. Move the newly-extracted `images/` folder to `/opt/bitnami/mediawiki/` to replace the folder deleted in step 1.

### Complete the restoration of content

Finally, run `php /opt/bitnami/mediawiki/maintenance/update.php` to complete the update. Then, restart the wiki: `sudo /opt/bitnami/ctlscript.sh restart`.

### Install RandomSelection

The `RandomSelection` extension is needed to support the random quote on the front page. To get it working:

1. Download the archive for the file [here](https://www.mediawiki.org/wiki/Special:ExtensionDistributor?extdistname=RandomSelection&extdistversion=REL1_41).
2. Extract it: `tar -xzf RandomSelection-REL1_41-d1bb390.tar.gz -C /opt/bitnami/mediawiki/extensions`
3. Add `wfLoadExtension( 'RandomSelection' );` to the last line of `/opt/bitnami/mediawiki/LocalSettings.php`.

## Configuring the domain name to display rather than the EC2 instance IP

Bitnami makes this easy: `/opt/bitnami/configure_app_domain --domain seventh-legion.net`. This script runs and then the domain displays properly when accessing the website.

## Configuring HTTPS

I'm following [this guide](https://docs.bitnami.com/aws/faq/administration/generate-configure-certificate-letsencrypt/). Basically, run `sudo /opt/bitnami/bncert-tool`. The first prompt is for a space-separated list of domains. I entered:

```
Domain list []: seventh-legion.net www.seventh-legion.net
```
From there, I just accepted the rest of the defaults. After the script completed, HTTPS appeared to work properly.

## Setting up the logo

I created a new logo locally as an SVG file (`sl_logo.svg`) and transferred it directly to `/opt/bitnami/mediawiki/resources/assets/` on the EC2 instance. Then, I updated these lines in `/opt/bitnami/mediawiki/LocalSettings.php`:

```
## The URL paths to the logo.  Make sure you change this from the default,
## or else you'll overwrite your logo when you upgrade!
$wgLogos = [
        '1x' => "$wgResourceBasePath/resources/assets/sl_logo.svg",
        'icon' => "$wgResourceBasePath/resources/assets/sl_logo.svg",
];
```