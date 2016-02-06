# DependencyRunner: Dependency Walker Reworked
## INTRO
DependencyRunner is yet another [Dependency Walker](http://www.dependencywalker.com/) implementation. 

The main goal of this project is Windows mechanisms study and application, and the Modern C++ practice as well. 

This implementation involves the following Windows mechanisms:

* I/O Completion Ports for fast file operations
* Memory mapping of PE headers for analysis
* Thread Pools

## HIGH LEVEL DESIGN
DependencyRunner builds an object model of the PE hierarchy for given PE file. This hierarchy is represented by a tree of PEModule instances.
The PEModule class represents a PE file. Since circular dependencies are possible, the model should be able to detect already existing nodes to avoid recurrency. 
To achieve that, the PEModule objects should be stored in a singleton repository, while references to siblings express the dependencies.

## IMPLEMENTATION DETAILS
The core system entities are _PEModuleRepository_ that holds the _PEModules_; every _PEModule_ exposes _API_ objects that represent functions that this module imports.
Every _API_ has reference to the _PEModule_ that exports it.

### DEPENDENCY SEARCH ALGORITHM
The dependency search process starts from the _rootModule_ -- the one provided by client. ATM, the _PEModuleRepository_ is empty.

* perform file path lookup by _parentModule_ name
* if the _PEModuleRepository_ already contains it, then return the existing _PEModule_ by path
* if the _PEModuleRepository_ does not contain it, then:
	1. the file is being read asynchronously using the I/O Completion Port
	2. on read complete, the PE header is mapped to memory
	3. _newPEModule_ is being created from this memory mapping as follows: 
		* read _parentModule_ manifest and activate the context by this manifest (save the previous context)
		* for each _dependencyModule_ in Imports and Delay Imports tables:
			* perform file path lookup by _dependencyModule_ name
			* if the path exists in _PEModuleRepository_, get the _dependencyPEModule_; otherwise, recurse this process for the _dependencyModule_ file path (step 1)
			* for each imported API in the _dependencyModule_:
				* read its name/ordinal
				* look for it in the _dependencyPEModule_ Exports table
				* if the _dependencyPEModule_ forwards it to another _forwardingModule_, recurse this process for the _forwardingModule_
				* store the API object (this object should also save additional data, e.g. is this a delay import, forwarded export, has undecorated name, etc)
		* deactivate the context
	4. insert the _newPEModule_ to the _PEModuleRepository_

Since the dependency search process is parallelized to multiple threads, the _PEModuleRepository_ has to be designed with concurrency considerations.
