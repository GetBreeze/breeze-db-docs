# Installation

To start using BreezeDb library either clone the [GitHub repository](https://github.com/GetBreeze/breeze-db) and copy the contents of the `src` folder over to your project's source folder or download the latest SWC file from the [releases section](https://github.com/GetBreeze/breeze-db/releases) and add it to your project's build path.

Verify that the library has been correctly installed by adding the following code and rebuilding your project:

```as3
import breezedb.BreezeDb;

trace("Using BreezeDb v" + BreezeDb.VERSION);
```

You should see a message with the version of the library you are using. At this point you are ready to [setup your database](../database/).