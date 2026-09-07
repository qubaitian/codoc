# Regenerate entire target files

Mapped blocks define the entire contents of their target files.  
codoc validates all mappings before deleting and recreating those files.  
This removes stale code outside mapped ranges on every generation.  
Handwritten code must live in files that the document does not target.  
