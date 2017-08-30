## Hashicorp's VAULT -POC Findings -

Hashicorp's vault can act as a middle man between different Security Providers.
It can abstract security token generation from Dynamic Backends like AWS.

### Concepts
Vault is a broker for other services and as such it can be setup to concentrate security facilitation.
It is setup as a cluster by default, so it is not a single point of failure in case things don't go well
in real life (when network disruptions happen, for instance).

Configuration for these services is mainly done in dialect of Ruby/JSON that is
called [HCL](https://github.com/hashicorp/hcl) (HashiCorp Configuration Language) 
which is basically JSON with comments and some other minor improvements.

### Dynamic Backends
For our purposes we could use Vault to add an extra layer of security to AWS's STS Federation and STS Assume Role Services.
These setups are configured in HCL by communicating with the Vault cluster. Then, when needed the Vault
can respond with temporary tokens that are managed by Vault and not by AWS even though they have the same fields
(plus some book keeping fields for Vault itself) as AWS would provide them.

### Vault API's
There are several API wrappers available, written in and are officially supported by HashiCorp
 * Go
 * Ruby

The community has created some more API libraries for languages like
 * Ansible
 * C#
 * Clojure
 * Rust
 * Scala
 
 just to name a few.
 
 The HashiCorp Vault API is basically a RESTful API using the http(s):// naming convention and the community has created
 wrappers to make it more programming interface alike.

### Tokens and Secrets
Tokens are presented in clear text to the consumer. This is not really an improvement over direct API communication with
the underlying services such as AWS.

### Conclusion
Personally, I don't see the real benefits in using Vault over AWS API communication.
It adds another layer on top of it and adds another tool to learn and master to our already complex toolset.
