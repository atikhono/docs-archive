---
layout: default
title: "Token-Based Authentication"
canonical: "/pe/latest/rbac_token_auth.html"
---

## Token-Based Authentication

### When Are Tokens Used?

In Puppet Enterprise (PE), users can generate tokens to authenticate their access to certain command line tools and API endpoints in PE. Authentication tokens can be used to manage access to the following PE services:

* [Puppet Orchestrator](./orchestrator_install.html#set-pe-rbac-permissions-and-token-authentication-for-puppet-orchestrator)
* [Code Manager](./code_mgr_webhook.html#request-an-rbac-token)
* [Node Classifier](./nc_forming_requests.html#authentication-token)
* [Role-Based Access Control (RBAC)](./rbac_forming_requests.html#authentication-token)
* [Activity](./rbac_activityapis.html#authentication-token)

Authentication tokens are tied to the [permissions granted to the user through RBAC](./rbac_permissions.html), and provide the user with the appropriate access to HTTP requests.

### Benefits of Using Tokens For Authentication

* Allows you to take action remotely, from your workstation for example, without having to log in to PE-managed servers.
* Tokens are RBAC aware, meaning that they reflect the permissions that a user has been granted through RBAC.
* Tokens are issued per user. Their usage is logged on a per-user basis. 
* Tokens have a limited lifetime.

### Generating a Token

The authentication tokens used in PE are JSON web tokens (JWT). They contain the following information:

* The time that the token was issued.
* The length of the token's lifetime.
* The user’s UUID.

You can generate a token using the API endpoint, or using the Puppet Access command line tool. Both of these methods are explained below. 

> **Note:** For security reasons, authentication tokens can only be generated for revocable users. The admin user and api_user cannot be revoked.

### Configuring Puppet Access

For information on installing the Puppet Access command line tool, bundled in the PE client tools package, see [Installing the PE Client Tools Package](./install_pe_client_tools.html).

After you've installed the PE client tools (either on the Puppet master or on a separate workstation), run the following command to create the Puppet Access configuration file:

~~~
mkdir ~/.puppetlabs
    echo '{"service-url":"https://<HOSTNAME>:4433/rbac-api"}' > ~/.puppetlabs/puppet-access.conf
~~~ 

The `service-url` setting in this Puppet Access config file points the tool at the RBAC service running on the Puppet console server.
 
#### Generating a Token Using the Command Line Tool

Puppet Access is a command line tool for generating and managing authentication tokens. 

To generate an authentication token using the Puppet Access command line tool:

1. From the command line on any PE-managed node, enter `puppet-access login`.

2. You will be prompted to enter a username and password. This is the same username and password that you use to log in to the PE console. 

   Tip: You can also pass your username as an argument, such as `puppet-access login <YOUR USER NAME>`. If you do this, you will only be prompted for a password.
   
3. Puppet Access contacts the [token endpoint in the RBAC API](./rbac_token.html). If your login credentials are correct, the RBAC service generates a token. 

4. The token is stored in a file for later use. The default location for storing the token is `~/.puppetlabs/token`.

> **Note:** If you run the login command with the `--debug` flag, the Puppet Access tool outputs the token, as well as the username and password. For security reasons, exercise caution when using the `--debug` flag with the login command.

> **Tip:** You can print the token at any time using `puppet-access show`.

#### Generating a Token Using the API Endpoint

The RBAC API includes a [token endpoint](./rbac_token.html) that you can use to generate a token. (The steps below assume that you are using cURL.) 

To generate a token:

1. Post your RBAC user login credentials. 

   In the command line, post your RBAC user login credentials using the token endpoint as shown below.

   `curl -k -X POST -H 'Content-Type: application/json' -d '{"login": "<YOUR PE USER NAME>", "password": "<YOUR PE PASSWORD>"}' https://<HOSTNAME>:4433/rbac-api/v1/auth/token`
   
   The parts of this cURL request are explained below:
   
   * `-k` flag: The `-k` flag turns off SSL verification of the RBAC server so that you can use the HTTPS protocol without providing a CA cert. Alternatively, you could use the `--cacert [FILE]` option and specify a CA certificate as described in [Forming Requests For the Node Classifier](./nc_forming_requests.html#authentication). If you do not provide one of these options in your cURL request, cURL will complain about not being able to verify the RBAC server.
   
   **Note:** The `-k` flag is shown as an example only. You should use your own discretion when choosing the appropriate server verification method for the tool that you are using. 
   
   * `-X POST`: This is an HTTP POST request to provide your login information to the RBAC service.
<<<<<<< HEAD
   * `-H 'Content-Type: application/json'`: sets the `Content-Type` header to `application/json` so that you receive a JSON-formatted response.
   * `-d '{"login": "<YOUR PE USER NAME>", "password": "<YOUR PE PASSWORD>"}'`: Provide the user name and password that you use to log in to the PE console. 
   * `https://<HOSTNAME>:4433/rbac-api/v1/auth/token`: Sends the request to the `token` endpoint. For `HOSTNAME`, provide the FQDN of the server that is hosting the PE console service. If you are making the call from the console server, you can use "localhost." The port number "4433" is the default port that PE services (node classifier service, RBAC service, and activity service) listen on. If you have changed the default port, provide the port that you are using.
   
2. Save the returned token.

   The returned token is a JSON object containing the key `token` and the token itself. Tokens are quite long. The following returned token example has been truncated:  
   
   ```
 {"token":"eyJhbGciOiJSUzUxMiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJhZG1pbiIsImlhdCI6MTQzODcyOTI1Niwic3ViIjp7ImxvZ2luIjoiYWRtaW4iLCJpZCI6IjQyYmYzNTFjLWY5ZWMtNDBhZi04NGFkLWU5NzZmZWM3ZjRiZCJ9fQ.KFMZEOiJ2cYV4FAY2UtLDEVlYlL5h1kCgyeKVbap3-oyi5OtXaYZc57xrusYG5k64xVMxIo9T9YPsKPbJM_3z5AypXCx6Y6nNWQk8vYC6fF2kklUzVKyXFerJWJJ6PipFMwzslsnPfKRDC-BjWclIU5ldegzIW9uYBcfGDBuDnzGQPeAsD7ttzos2lQwC0..."}
   ```
       
   Copy the returned token to a text file or environment variable. For example, you can save it as an environment variable using `export TOKEN=<PASTE THE TOKEN HERE>`.
   
#### Generating a Token For Use By a Service

If you need to generate a token for use by a service and the token doesn't need to be saved, use the `--print` option with the Puppet Access command line tool. `puppet-access login [username] --print` will generate a token, and then display the token content as stdout (standard output) rather than saving it to disk. 

When generating a token for a service, you may want to specify a longer token lifetime so that you don't have to regenerate the token too frequently. See [Setting a Token-Specific Lifetime](#setting-a-token-specific-lifetime).

### Using a Token

When you have generated the token, you can continue to use it until it expires, or your access is revoked. The token will have the exact same set of permissions as the user that generated it.

#### Using a Token With PE API endpoints

The example below shows how to use a token in an API request. In this example, we are using the `/users/current` endpoint of the RBAC API to get information about the current authenticated user. The example assumes that you saved your token as an environment variable using `export TOKEN=<PASTE THE TOKEN HERE>`.

   `curl -k -x GET https://<HOSTNAME>:4433/rbac-api/v1/users/current -H "X-Authentication:$TOKEN"`
   
The example above uses the X-Authentication header to supply the token information. In some cases, such as GitHub webhooks, you may need to supply the token in a token parameter. To supply the token in a token parameter, you would specify the request as follows:

   `curl -k -X GET https://<HOSTNAME>:4433/rbac-api/v1/users/current?token=$TOKEN` 
   
   **Note:** Supplying the token as a token parameter is not as secure as using the X-Authentication method.

### Viewing Token Activity

Token activity is logged by the PE activity service. 

To view token generation activity, in the PE console, click **Access control** > **Users**, and click on the user that you are interested in. In the user details screen, click the **Activity** tab. 

### Managing Tokens

#### Changing the Default Lifetime

Tokens have a default lifetime of five minutes. To change the default period, in the PE console, click **Nodes** > **Classification** > **PE Console**. In the classes tab, find the `puppet_enterprise::profile::console` class. In the **Parameter name** dropdown, select the `rbac_token_auth_lifetime` parameter. In **Value**, enter the lifetime value. You can specify a numeric value followed by "y" (years), "d" (days), "h" (hours), "m" (minutes), or "s"(seconds). For example, a value of "12h" will be 12 hours. Do not add a space between the numeric value and the unit of measurement. If you do not specify a unit, it is assumed to be seconds.

> **Note:** If you do not want the token to expire, set the lifetime to "0". Setting it to zero gives the token a lifetime of approximately ten years.

> **Note:** Any user with permission to edit infrastructure-related node groups will be able to change the default lifetime setting. To restrict access, see the recommendation in [Managing user access to the lifetime setting for authentication tokens](./rbac_permissions.html#managing-user-access-to-the-lifetime-setting-for-authentication-tokens).

#### Setting a Token-Specific Lifetime

When generating a token, you can change the lifetime of the token if you have the ["Tokens - Override default lifetime" permission](./rbac_permissions.html#available-permissions). This permission is given to the Administrators and Operators roles by default. You can remove the permission from the Operators role.

> **Note:** If a user doesn't have this permission, but they have the permission to edit the PE Console node group, they will still be able to change the default lifetime setting for all tokens through the `rbac_token_auth_lifetime` parameter. To avoid unintentionally granting excessive permissions, see the note in [Managing user access to the lifetime setting for authentication tokens](./rbac_permissions.html#managing-user-access-to-the-lifetime-setting-for-authentication-tokens).

**When generating a token using the command line tool:**

When generating a token using the Puppet Access command line tool, you can use the `lifetime` option to specify the amount of time for which the token is valid. For example, `puppet-access login --lifetime 1h`. 

**When generating a token using the API endpoint:**

When generating a token using [the `token` endpoint of the RBAC API](./rbac_token.html), you can use the `lifetime` value to specify the amount of time for which the token is valid. For example, `{"login": "<YOUR PE USER NAME>", "password": "<YOUR PE PASSWORD>", "lifetime": "1h"}`.

>**Note:** With the exception of the UUID, the user information in a token is subject to change. If user information changes in your user directory, the token will become invalid.

#### Deleting a Token File

If you logged in to the Puppet Access command line tool to generate a token, you can remove the file that stores that token simply by running the `delete-token-file` command. This is useful if you are working on a server that is used by other people. Removing the token file prevents other users from using your authentication token, but it does not actually revoke the token. 

To delete your token file, from the command line, type `puppet-access delete-token-file`.

If you do not specify a path, the tool will look for the token file at the default path. If you have used a different path to store your token file, specify `--token-path` as follows:

   `puppet-access delete-token-file --token-path <YOUR TOKEN PATH>`


#### Revoking a Token

To revoke a token, you need to revoke the access of the user who generated the token. The user's access needs to be revoked until the token expires. If you are using a token with a very long expiration period, consider creating a special user for that token so that the user can be revoked.

To revoke a user:

1. In the right navigation bar of the PE console, click **Access control**.

2. Click **Users** and click the user that you want to revoke.

3. At the upper right of the screen, click **Revoke user access**. 

#### Changing Configuration Settings

There are several options that you can specify with the `puppet-access` command in the command line. 

* `--service-url`: Changes the contact information for the server that is issuing the token. The token is issued by the RBAC service. You would typically only need to change this if you are moving the PE console server. 

  (Example: `puppet-access --service-url https://consoleserver.example.com:4433/rbac-api`)

* `-c`, `--config-file`: Changes the location of your configuration file.
  
  (Default: ~/.puppetlabs/puppet-access.conf) 

* `-t`, `--token-file`: Changes the location of the file that the token is stored in.

  (Default: ~/.puppetlabs/token) 

### Security Best Practices For Using Token Authentication

1. Use the minimum required lifetime value 
	
	If a token is stolen, the only way to revoke access to the token currently is to revoke the associated user account for the remaining duration of the token's lifetime. To avoid having to revoke a user for an excessively long time, it's a good practice to set the lifetime of a token to the minimum required value. 
	
	If you need to have a token with a long lifetime, [use a certificate and whitelist](/references/4.3.latest/man/cert.html) rather than a token so that you can easily disable it at anytime by removing it from the certificate whitelist. For more information about whitelisting and using certificates, see [Authentication](./nc_forming_requests.html#authentication).
	
2. Store credentials with care
	
	* Restrict access to files that store tokens or certificates. Make sure only people who should be using those credentials can read the files.
	
	* Don’t save your password, especially not in a script.
	
3. Don't accidentally expose credentials
	
	* If you use query parameters for your tokens (when using Code Manager for example), unencrypted passwords can get logged by various tools.
	
	* In Puppet Access, your password is output when you log in using the debug flag.
	
4. Using cURL -k
	
	If you are using cURL with the `-k` insecure SSL connection option, keep in mind that you are vulnerable to a person-in-the-middle attack. 
	
5. Revoke or remove external directory users
	
	If a remote user generates a token and then that user is deleted from your external directory service, the deleted user cannot log in to the console. However, since the token has already been authenticated, the RBAC service does not contact the external directory service again when the token is used in the future. To fully remove the token's access, you need to manually revoke or delete the user from PE.
