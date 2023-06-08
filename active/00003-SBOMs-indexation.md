The SBOMS prodsec releases are for products and container images. 
Products are referenced by a `CPE`, and images with a `purl`.

Application-level products are better represented with a product identifier on the top level
just to provide a common abstraction for the set of components identified.

Taking a look at prodSec SPDXes files, there are thousands of components in a product.
So we have the following relationships :
```
SBOM<CPE/purl>  -- 1:n -->  Components<purl>
A               
|1:n
|     
VEX <--1:N-- CVE   
```

Currently bombastic stores SBOMS identified with a purl (any string really, but we assume it is purl).
we need to store SBOMS with a key that can reference a purl OR a CPE. 

Food for thought :
 - it's not really that important what the key is as long as it's unique.
 - It needs to be a single value (a composite tuple does not do it)
 - `purl` and `cpe` have identifiers at the beginning : `cpe:/a:redhat:amq_interconnect:1::el7`, `purl://pkg:npm/relateurl@0.2.7`
 but we may need to add the `purl` header manually as it's not in the prodsec SBOMs
 - a prodsec SBOM can contain multiple CPE at the root level ?
 - 
