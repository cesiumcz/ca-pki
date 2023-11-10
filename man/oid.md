## OID request
OID (*Object Identifier*) is globally-unique number that is, in PKI context, used in X.509v3 to refer:
- [X.509v3 extensions](../openssl.cnf.templ.clean#L116), including:
- **CPS** (Certificate Practice Statement)

**We wont create a new X.509v3 extension, but we will need a unique OID to refer our CPS (certificate policy / policies).**

### Obtain OID
If our organization does not have global unique OID yet, we can request for OID for free at [IANA Application page](https://www.iana.org/assignments/enterprise-numbers/assignment/apply/).  
Once acquired, our OID namespace will look as follows: `1.3.6.1.4.1.{PEN}`, 1.3.6.1.4.1 being IANA's root for Private Enterprise numbers tree.  
The arrangement and maintenance of our OID subtree is entirely up to us.  
We can create a node for CA (1), node for CA ID `{CA_ID}` and a node for certificate policy statement `{CPS}` (may start with 1). CPS OID will then look like `1.3.6.1.4.1.{PEN}.1.{CA_ID}.{CPS}`.  
CPS should be published on URI which will be then [embedded in a certificate](../openssl.cnf.templ.clean#L202) alongside with CPS OID.
