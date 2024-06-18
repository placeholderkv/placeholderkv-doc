Returns the scripts in the script cache.

This command accepts one SHA1 digest and return a script if the script
are already defined or NOSCRIPT error if not inside the script cache.
This can be useful to debug. In some scenarios, the business may not be able to find the
previously used Lua script and only have a SHA signature.
Or there are multiple identical evalsha's args in monitor/slowlog,
and admin is not able to distinguish the script body.

For more information about `EVAL` scripts please refer to [Introduction to Eval Scripts](../topics/eval-intro.md).
