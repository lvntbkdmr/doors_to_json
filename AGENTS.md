
This is a DXL project for DOORS 9.5

It takes a module path as an argument and row by row it iterates every object within that module and export them into json file.

There can be links into other modules from that input module, if that is the case for any object, export that module into json too.

Additionally, there should be relations between objects within json.

Because in the end, those json data will be fed into a python script that will create/update sqlite database based on that.

Relations between objects can be many-to-many


