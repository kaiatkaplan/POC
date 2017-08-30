### Synopsis
	Vault is a tool for securely accessing secrets.
	A secret is anything that you want to tightly control access to,
	such as API keys, passwords, certificates, and more. Vault provides a
	unified interface to any secret, while providing tight
	access control and recording a detailed audit log.

### URL
	This is the main URL for [Vault](https://www.vaultproject.io) by Hashicorp.

### Tests ###
	Followed the [Getting Started](https://www.vaultproject.io/intro/getting-started/install.html) guide on the website
	to setup and install Hashicorp's Vault in a [Docker container](https://hub.docker.com/_/vault/).
	Followed the instructions to install and run Vault locally.  For this POC I used
	*brew install vault*

### API Access
	[HTTP Access](https://www.vaultproject.io/intro/getting-started/apis.html) is done with standard RESTful calls.
	curl can be used for it or any programming language that can speak RESTfully.

### Observations
	The tool is fairly easy to use when installed locally.
	Setting, Getting and Manipulating keys and values were simple and straight
	forward.  The syntax is clear and concise.

### Docker integration
	The image installed and ran just fine in a container when I followed the
	aforementioned Getting Started guide.  In my first trials I was not able to
	set/get values using the docker installed vault.  A second test will need to be
	done.

### Pros

	- [x] AWS support
	
	- [x] GitHub support
	
	- [x] Simple interface to complex structures, like AWS groups/users and roles
	 
### Cons
	
	- [ ] Another tool to learn
	
	- [ ] Another gem to use
	

## Detailed Setup Procedure for a Vault Server a SSH client and a MySQL Client

Follow this [Link](SETUP.md) to get to the document.
