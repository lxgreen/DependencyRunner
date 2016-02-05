DependencyRunner: Dependency Walker Reworked
============================================

INTRO
-----

DependencyRunner is yet another [Dependency Walker](http://www.dependencywalker.com/) implementation. 

The main goal of this project is Windows mechanisms study and application, and the Modern C++ practice as well. 

This implementation involves the following Windows mechanisms:

* I/O Completion Ports for fast file operations
* Memory mappings of PE headers for analysis
* Thread Pools

HIGH LEVEL DESIGN
-----------------

DependencyRunner builds an object model of the PE hierarchy for any given PE file. This hierarchy is represented by a tree of PEModule instances.
The PEModule class represents a PE file. It exposes the relevant PE header sections such as Imports table, Exports table, Delay Imports, etc.
Since circular dependencies are possible, the model should be able to detect already existing nodes and avoid recurrency. 
To achieve that, the PEModule objects should be stored in a singleton repository, and hold references to siblings to express dependencies.

IMPLEMENTATION DETAILS
----------------------

The core system entities are _PEModuleRepository_ that holds the _PEModules_; every _PEModule_ exposes _API_ objects that represent functions that this module imports.
Every _API_ has reference to the _PEModule_ that exports it.

__DEPENDENCY SEARCH__

The first step is to find the proper file for the given module. The file search involves both system and user directories search; SxS and API Schema should be considered.

If the file is found, there are 2 options: 

* if the _PEModuleRepository_ already contains it, then the existing _PEModule_ is returned
* if the _PEModuleRepository_ does not contain it, then:
	* the file is being read asynchronously using the I/O Completion Port
	* on read complete, the PE header is mapped to memory
	* new _PEModule_ is being created from this memory mapping, and added to the _PEModuleRepository_ by the file path as the repository key

Since the dependency search process is parallelized to multiple threads, the _PEModuleRepository_ has to be designed with concurrency considerations.

The dependency search process starts from the _root_ module -- the one provided by client. ATM, the _PEModuleRepository_ is empty.


__PEMODULE INITIALIZATION__



