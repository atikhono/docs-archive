---
layout: default
title: "Using an External Certificate Authority with Puppet Enterprise"
canonical: "/pe/latest/external_ca.html"
---

The different parts of Puppet Enterprise (PE) use SSL certificates to communicate securely with each other. PE uses its own certificate authority (CA) to generate and verify these credentials.

However, you may already have your own CA in place and wish to use it instead of PE's integrated CA. This page will familiarize you with the certificates and security credentials signed by the PE CA, then detail the procedures for replacing them.

## Before You Begin

Setting up an external certificate authority (CA) to use with PE is beyond the scope of this document; in fact, this writing assumes that you already have some knowledge of CA and security credential creation and have the ability to set up your own external CA. This document will lead you through the certs and security credentials you'll need to replace in PE. However, before beginning, we recommend you familiarize yourself with the following docs:

- [SSL Configuration: External CA Support](/puppet/4.3/reference/config_ssl_external_ca.html) provides guidance on establshing an external CA that will play nice will Puppet (and therefore PE).

- [Puppet Server: External CA Configuration](/puppetserver/2.2/external_ca_configuration.html) details configurations for Puppet Server when using an external CA.

>**Note:** PE does not support the use of intermediate CAs.

### What You Will Need

Before you get started, you should have the following:

- The **complete** X.509 Certificate Authority certificate chain for the external party CA, in PEM format

- An X.509 Certificate, signed by the external party CA, in PEM format, with matching private and public keys, for each PE service which uses a certificate

- An X.509 Certificate, signed by the external party CA, in PEM format, with matching private and public keys, for each Puppet agent node

- Run `puppet cert list --all` on your Puppet master server to inspect the inventory of certificates signed using PE's built-in CA. You will need a cert, private key, and public key for each.

   -  Per-node certificates for the Puppet master (and any agent nodes)`
   - `pe-internal-classifier`
   - `pe-internal-console-mcollective-client`
   - `pe-internal-dashboard`
   - `pe-internal-mcollective-servers`
   - `pe-internal-peadmin-mcollective-client`
   
   Each of these will need to be replaced with new certificates signed by your external CA. The steps below will explain how to find and replace these credentials.
   
- For **each certificate you create**, you need to add the following information:
   - For the X.509 extension, `Extended Key Usage`, add `serverAuth` and `clientAuth`.
   - Comment out the entry, `nsCertType=server`.
   - Add any Puppet master DNS alt names to the cert. Note that the default alt name `puppet` is always included. 
   
>**Important: Shared Certificate and Security Credentials**
>
>In Puppet Enterprise, the Puppet master and the Puppet agent services share the same certificate, so replacing the shared certificate will suffice for both services. In other words, if you replace the Puppet master certificate, you don't need to separately replace the agent certificate.

## Replacing the PE Certificate Authority and Security Credentials

> **Important**: For ease of use, we recommend naming *ALL* of your certificate and security credential files exactly the same way they are named by PE and replace them as such on the Puppet master; for example, use the `cp` command to overwrite the file contents of the certs generated by the PE CA. This will ensure that PE will recognize the file names and not overwrite any files when you perform Puppet runs. In addition, this will prevent you from needing to touch various config files, and thus, limit the chances of problems arising.
>
> The remainder of this doc assumes you will be using identical files names.

We recommend that once you've set up your external CA and security credentials, you first replace the files for PE master/agent nodes and the PE console, then replace the files for PuppetDB, and then replace the PE MCollective files. Remember, naming the new certs and security credentials exactly as they're named by PE will ensure the easiest path to success.

### Step 1: Replace the PE Master and PE Console Certificates and Security Credentials

1. On the Puppet master, clear the cached catalog.

   ~~~
   rm -f /opt/puppetlabs/puppet/cache/client_data/catalog/<PUPPET MASTER FQDN>.json
   ~~~

2. From the console, click **Nodes** > click **Classification**.
3. Click the **PE Certificate Authority** group.
4. From the **PE Certificate Authority** group page, find the Puppet master (under **Certname**) and click **unpin**.
5. Click the **Commit change** button.
6. Navigate the **PE Master** group page and click the **Classes** tab.
7. Locate the `puppet_enterprise::profile::master` class, and from the **Parameter** drop-down list, select **enable_ca_proxy**.
8. Set the **enable_ca_proxy** parameter value to **false**.
9. Click **Add parameter**, and then click the **Commit change** button.
10. Kick off a Puppet run, with `puppet agent -t`.
11. Stop all PE services.

    ~~~
    puppet resource service puppet ensure=stopped
    puppet resource service pe-puppetserver ensure=stopped
    puppet resource service pe-activemq ensure=stopped
    puppet resource service mcollective ensure=stopped
    puppet resource service pe-puppetdb ensure=stopped
    puppet resource service pe-postgresql ensure=stopped
    puppet resource service pe-console-services ensure=stopped
    puppet resource service pe-nginx ensure=stopped
    puppet resource service pe-orchestration-services ensure=stopped
    ~~~

12. Copy your new certs and security credentials for the Puppet master to the relevant locations.

    |External CA Generated File |Location on Host |Name of File|
    |---|---|---|---|
    |CA's certificate|/etc/puppetlabs/puppet/ssl/certs|ca.pem|
    |CA's CRL|/etc/puppetlabs/puppet/ssl|crl.pem|
    |CA's CRL|/etc/puppetlabs/puppet/ssl/ca|ca_crl.pem|
    |Host's Cert|/etc/puppetlabs/puppet/ssl/certs|\<PUPPET MASTER FQDN>.pem	|
    |Host's Public Key|etc/puppetlabs/puppet/ssl/public_keys|\<PUPPET MASTER FQDN>.pem|
    |Host's Private Key|etc/puppetlabs/puppet/ssl/private_keys|\<PUPPET MASTER FQDN>.pem|

    <br>

    ~~~
    cp <PATH TO NEW PUPPET MASTER FQDN>.cert /etc/puppetlabs/puppet/ssl/certs/<PUPPET MASTER FQDN>.pem
    cp <PATH TO NEW PUPPET MASTER FQDN>.private_key /etc/puppetlabs/puppet/ssl/private_keys/<PUPPET MASTER FQDN>.pem
    cp <PATH TO NEW PUPPET MASTER FQDN>.public_key /etc/puppetlabs/puppet/ssl/public_keys/<PUPPET MASTER FQDN>.pem
    cp <PATH TO NEW CA CRL>.pem /etc/puppetlabs/puppet/ssl/crl.pem
    cp <PATH TO NEW CA CRL>.pem /etc/puppetlabs/puppet/ssl/ca/ca_crl.pem
    cp <PATH TO NEW CA CRT>.pem /etc/puppetlabs/puppet/ssl/certs/ca.pem
    ~~~

13. Copy your new certs and security credentials for the PE console to the relevant locations.

    On a split install these will be on the console server.

    | External CA Generated File | Location on Host | Name of File | 
    |---|---|---|---|
    | Internal-dashboard cert   | /opt/puppetlabs/server/data/console-services/certs/  | pe-internal-dashboard.cert.pem  |   
    | Internal-dashboard private key   | /opt/puppetlabs/server/data/console-services/certs/  | pe-internal-dashboard.private_key.pem  |
    |Internal-dashboard public key  | /opt/puppetlabs/server/data/console-services/certs/   | pe-internal-dashboard.public_key.pem  |

    <br>

    ~~~
    cp <PATH TO NEW PE-INTERNAL-DASHBOARD>.cert /opt/puppetlabs/server/data/console-services/certs/pe-internal-dashboard.cert.pem
    cp <PATH TO NEW PE-INTERNAL-DASHBOARD>.private_key /opt/puppetlabs/server/data/console-services/certs/pe-internal-dashboard.private_key.pem
    cp <PATH TO NEW PE-INTERNAL-DASHBOARD>.public_key /opt/puppetlabs/server/data/console-services/certs/pe-internal-dashboard.public_key.pem
    ~~~

### Step 2: Replace the PE-Internal-Orchestration Certificate and Security Credentials

1. Using the same external CA, generate a certificate and security credentials for the internal orchestration services.

   On a split install these will be on the Puppet master server.

   | External CA Generated File | Location on Host | Name of File | 
   |---|---|---|---|
   | Internal-orchestrator cert  | /etc/puppetlabs/puppet/ssl/certs/   | pe-internal-orchestrator.pem  | 
   | Internal-orchestrator private key  | /etc/puppetlabs/puppet/ssl/certs/private_keys  | pe-internal-orchestrator.pem |
   | Internal-orchestrator public key  | /etc/puppetlabs/puppet/ssl/certs/public_keys  | pe-internal-orchestrator.pem  | 

   <br>

   ~~~
   cp <PATH TO NEW PE-INTERNAL-ORCHESTRATOR>.cert /etc/puppetlabs/puppet/ssl/certs/pe-internal-orchestrator.cert.pem
   cp <PATH TO NEW PE-INTERNAL-ORCHESTRATOR>.public_key /etc/puppetlabs/puppet/ssl/public_keys/pe-internal-orchestrator.public_key.pem
   cp <PATH TO NEW PE-INTERNAL-ORCHESTRATOR>.private_key /etc/puppetlabs/puppet/ssl/private_keys/pe-internal-orchestrator.private_key.pem
   ~~~

2. Continue to step 3 to replace the internal classifier certificate and security credentials.

### Step 3: Replace the Internal Classifier Certificate and Security Credentials

Use the same certs and creds for the Puppet master as you did in step 1. The pe-internal-classifier certs and keys are new.

1. Using the same external CA, generate a certificate and security credentials for the internal classifier (i.e., the node classifier).

   On a split install these will be on the console server.

   | External CA Generated File | Location on Host | Name of File | 
   |---|---|---|---|
   | Internal-classifier cert  | /opt/puppetlabs/server/data/console-services/certs/   | pe-internal-classifier.cert.pem  | 
   | Internal-classifier private key  | /opt/puppetlabs/server/data/console-services/certs/  | pe-internal-classifier.private_key.pem |
   | Internal-classifier public key  | /opt/puppetlabs/server/data/console-services/certs/  | pe-internal-classifier.public_key.pem  | 

   <br>

   ~~~
   cp <PATH TO NEW PE-INTERNAL-CLASSIFIER>.cert /opt/puppetlabs/server/data/console-services/certs/pe-internal-classifier.cert.pem
   cp <PATH TO NEW PE-INTERNAL-CLASSIFIER>.public_key /opt/puppetlabs/server/data/console-services/certs/pe-internal-classifier.public_key.pem
   cp <PATH TO NEW PE-INTERNAL-CLASSIFIER>.private_key /opt/puppetlabs/server/data/console-services/certs/pe-internal-classifier.private_key.pem
   ~~~

2. In this same directory (i.e., `/opt/puppetlabs/server/data/console-services/certs/`) add the new certs and security credentials for the Puppet master.

   >**Note**: For a split install these are the Puppet agent certs, found in the same locations as the Puppet master certs, i.e.,
   >
   >* `/etc/puppetlabs/puppet/ssl/certs/<AGENT FQDN>.pem`
   >* `/etc/puppetlabs/puppet/ssl/private_keys/<AGENT FQDN>.pem`
   >* `/etc/puppetlabs/puppet/ssl/public_keys/<AGENT FQDN>.pem`

   <br>

   ~~~
   cp /etc/puppetlabs/puppet/ssl/certs/<PUPPET MASTER FQDN>.pem /opt/puppetlabs/server/data/console-services/certs/<PUPPET MASTER FQDN>.cert.pem
   cp /etc/puppetlabs/puppet/ssl/private_keys/<PUPPET MASTER FQDN>.pem /opt/puppetlabs/server/data/console-services/certs/<PUPPET MASTER FQDN>.private_key.pem
   cp /etc/puppetlabs/puppet/ssl/public_keys/<PUPPET MASTER FQDN>.pem /opt/puppetlabs/server/data/console-services/certs/<PUPPET MASTER FQDN>.public_key.pem
   ~~~

### Step 4: Replace the Puppet Master Certificate and Security Credentials in PostgreSQL

1. Add the Puppet master's certificate and security credentials to the PostgreSQL certs directory, located at `/opt/puppetlabs/server/data/postgresql/9.4/data/certs/`.

~~~
cp /etc/puppetlabs/puppet/ssl/certs/<PUPPET MASTER FQDN>.pem /opt/puppetlabs/server/data/postgresql/9.4/data/certs/<PUPPET MASTER FQDN>.cert.pem
cp /etc/puppetlabs/puppet/ssl/private_keys/<PUPPET MASTER FQDN>.pem /opt/puppetlabs/server/data/postgresql/9.4/data/certs/<PUPPET MASTER FQDN>.private_key.pem
cp /etc/puppetlabs/puppet/ssl/public_keys/<PUPPET MASTER FQDN>.pem /opt/puppetlabs/server/data/postgresql/9.4/data/certs/<PUPPET MASTER FQDN>.public_key.pem
~~~

### Step 5: Replace the PuppetDB Certificates and Security Credentials

1. Replace PuppetDB's certs and security credentials.

   > **Note**: For a split install these are the Puppet agent certs, found in the same locations as the Puppet master certs, i.e.,
   >
   >* `/etc/puppetlabs/puppet/ssl/certs/<AGENT FQDN>.pem`
   >* `/etc/puppetlabs/puppet/ssl/private_keys/<AGENT FQDN>.pem`
   >* `/etc/puppetlabs/puppet/ssl/public_keys/<AGENT FQDN>.pem`
   <br>

   | External CA Generated File | Location on Host | Name of File | Notes |
   |---|---|---|---|
   | PuppetDB cert  | /etc/puppetlabs/puppetdb/ssl  | /etc/puppetlabs/puppetdb/ssl/\<PUPPET MASTER CERT>.pem | On a monolithic install, copy the Puppet master's agent cert to this location; on split install, copy the Puppet agent's cert to this location   |
   | PuppetDB private key  | /etc/puppetlabs/puppet/ssl  | /etc/puppetlabs/puppetdb/ssl/\<PUPPET MASTER PRIVATE KEY>.pem| On a monolithic install, copy the Puppet master's agent cert to this location; on split install, copy the Puppet agent's cert to this location   |
   | PuppetDB public key  | /etc/puppetlabs/puppet/ssl  | /etc/puppetlabs/puppetdb/ssl/\<PUPPET MASTER PUBLIC KEY>.pem  | On a monolithic install, copy the Puppet master's agent cert to this location; on split install, copy the Puppet agent's cert to this location   |

   <br>

   ~~~
   cp /etc/puppetlabs/puppet/ssl/certs/<PUPPET MASTER FQDN>.pem /etc/puppetlabs/puppetdb/ssl/<PUPPET MASTER FQDN>.cert.pem
   cp /etc/puppetlabs/puppet/ssl/private_keys/<PUPPET MASTER FQDN>.pem /etc/puppetlabs/puppetdb/ssl/<PUPPET MASTER FQDN>.private_key.pem
   cp /etc/puppetlabs/puppet/ssl/public_keys/<PUPPET MASTER FQDN>.pem /etc/puppetlabs/puppetdb/ssl/<PUPPET MASTER FQDN>.public_key.pem
   ~~~

### Step 6: Replace the MCollective Certificates and Security Credentials

1. Generate new credentials for each McCollective cert, then replace the cert, private key, and public key for each of them.

   | External CA Generated File | Location on Host | Name of File | 
   |---|---|---|---|
   | MCollective internal servers cert  | /etc/puppetlabs/puppet/ssl/certs  | pe-internal-mcollective-servers.pem	  |   
   | MCollective internal servers priv key  | /etc/puppetlabs/puppet/ssl_private_keys  | pe-internal-mcollective-servers.pem	  |   
   | MCollective internal servers pub key  | /etc/puppetlabs/puppet/ssl_public_keys  | pe-internal-mcollective-servers.pem  |   
   | MCollective client cert | /etc/puppetlabs/puppet/ssl/certs  | pe-internal-peadmin-mcollective-client.pem  |   
   | MCollective client priv key | /etc/puppetlabs/puppet/ssl_private_keys  | pe-internal-peadmin-mcollective-client.pem  |   
   | MCollective client pub key | 	/etc/puppetlabs/puppet/ssl_public_keys  | pe-internal-peadmin-mcollective-client.pem	  |   
   | MCollective internal console client cert  | /etc/puppetlabs/puppet/ssl/certs  | pe-internal-puppet-console-mcollective-client.pem
   | MCollective internal console client priv key | /etc/puppetlabs/puppet/ssl_private_keys  | pe-internal-puppet-console-mcollective-client.pem |
   | MCollective internal console client pub key | /etc/puppetlabs/puppet/ssl_public_keys  | pe-internal-puppet-console-mcollective-client.pem |

   <br>

   ~~~
   cp <PATH TO NEW PE-INTERNAL-MCOLLECTIVE-SERVERS>.cert /etc/puppetlabs/puppet/ssl/certs/pe-internal-mcollective-servers.pem
   cp <PATH TO NEW PE-INTERNAL-MCOLLECTIVE-SERVERS>.private_key /etc/puppetlabs/puppet/ssl/private_keys/pe-internal-mcollective-servers.pem
   cp <PATH TO NEW PE-INTERNAL-MCOLLECTIVE-SERVERS>.public_key /etc/puppetlabs/puppet/ssl/public_keys/pe-internal-mcollective-servers.pem
   cp <PATH TO NEW PE-INTERNAL-MCOLLECTIVE-CLIENT>.cert /etc/puppetlabs/puppet/ssl/certs/pe-internal-peadmin-mcollective-client.pem
   cp <PATH TO NEW PE-INTERNAL-MCOLLECTIVE-CLIENT>.private_key /etc/puppetlabs/puppet/ssl/private_keys/pe-internal-peadmin-mcollective-client.pem
   cp <PATH TO NEW PE-INTERNAL-MCOLLECTIVE-CLIENT>.public_key /etc/puppetlabs/puppet/ssl/public_keys/pe-internal-peadmin-mcollective-client.pem
   cp <PATH TO NEW PE-INTERNAL-PUPPET-CONSOLE-MCOLLECTIVE-CLIENT>.cert /etc/puppetlabs/puppet/ssl/certs/pe-internal-puppet-console-mcollective-client.pem
   cp <PATH TO NEW PE-INTERNAL-PUPPET-CONSOLE-MCOLLECTIVE-CLIENT>.private_key /etc/puppetlabs/puppet/ssl/private_keys/pe-internal-puppet-console-mcollective-client.pem
   cp <PATH TO NEW PE-INTERNAL-PUPPET-CONSOLE-MCOLLECTIVE-CLIENT>.public_key /etc/puppetlabs/puppet/ssl/public_keys/pe-internal-puppet-console-mcollective-client.pem
   ~~~


2. On the Puppet master, navigate to `/etc/puppetlabs/activemq/`.
3. Remove the following two files: `broker.ts` and `broker.ks`.

### Step 7: Edit `puppet.conf`

1. Navigate to `/etc/puppetlabs/puppet/` and open `puppet.conf`.
2. In the `[agent]` section add, `certificate_revocation=false`

### Step 8: Restart Services and Run Puppet

At this point, you need to restart or start the various services that help control Puppet and then run Puppet so that the certs and security credentials can be properly distributed.

1. Restart all PE services.

   ~~~
   puppet resource service puppet ensure=running
   puppet resource service pe-puppetserver ensure=running
   puppet resource service pe-activemq ensure=running
   puppet resource service mcollective ensure=running
   puppet resource service pe-puppetdb ensure=running
   puppet resource service pe-postgresql ensure=running
   puppet resource service pe-console-services ensure=running
   puppet resource service pe-nginx ensure=running
   puppet resource service pe-orchestration-services ensure=running
   ~~~


2. From the Puppet master node, use the command line to kick off a Puppet run with `puppet agent -t`.

During this run, Puppet will copy the credentials you replaced into their final locations, regenerate the ActiveMQ truststore and keystore, and restart the `pe-activemq` and `mcollective` services.

To ensure the certs have been properly distributed, got so the **Node Inventory** page in the console and click the Puppet master node in the list. Then, on the **Reports** tab, review the information for the node.

## Adding Agent Nodes Using Your External CA

1. Install Puppet Enterprise on the node, if it isn't already installed.
2. Using the same external CA you used for the Puppet master, create a cert and private key for your agent node.
3. Locate the files you will need to replace on the agent. Refer to  [Locating the PE Agent Certificate and Security Credentials](#locating-the-puppet-agent-certificate-and-security-credentials) to find them.
4. Copy the agent's certificate, private key, and public key into place. Do the same with the external CA's CRL and CA certificate.
5. Restart the `puppet` service.

Your node should now be able to do Puppet agent runs, and its reports will appear in the console. (You can accelerate this by letting Puppet run once, waiting a few minutes for the node to be added to the MCollective group in the console, and then running `puppet agent -t`.)

If you still don't see your agent node in **Node Inventory**, use NTP to verify that time is in sync across your PE deployment. (You should *always* do this anyway.)

### Locating the Puppet Agent Certificate and Security Credentials

Every system under PE management (including the Puppet master, console, and PuppetDB) runs the Puppet agent service. To determine the proper locations for the certificate and security credential files used by the Puppet agent, run the following commands:

- Certificate: `puppet agent --configprint hostcert`
- Private key: `puppet agent --configprint hostprivkey`
- Public key: `puppet agent --configprint hostpubkey`
- Certificate Revocation List: `puppet agent --configprint hostcrl`
- Local copy of the CA's certificate: `puppet agent --configprint localcacert`
