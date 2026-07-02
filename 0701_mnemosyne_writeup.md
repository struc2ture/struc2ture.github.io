There are two parts to this story: a prologue and an epilogue. One, an experiment with relocatable memory. Two, a learning project to implement my own malloc. The result is an experimental library, Mnemosyne.

# One. Relocatable Memory & Relative Pointers

When working on games I've often questioned why it's so complicated to implement even the simplest version of save files. Immediately you need to bring in complex serialization libraries or hand roll your own reflection to traverse the state and preserve only the data that needs to be saved to disk. Then there's versioning. Of course you will need these things for a mature project. But why do you have to bear this complexity upfront, even in the early stages, when it's most important for the codebase to not limit experimentation and exploration?
