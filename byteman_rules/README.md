## Get Byteman

 * http://downloads.jboss.org/byteman/3.0.10/byteman-download-3.0.10-bin.zip
 * unzip, in this example: ```/home/karm/Tools/```

### Env vars
This example uses the undermentioned env vars:
```
export BYTEMAN_HOME=/home/karm/Tools/byteman-download-3.0.10
export PATH=${PATH}:${BYTEMAN_HOME}/bin
export MY_BTM_RULES=/home/karm/Skola/podzim2017/PA193/PA193_SixEleven_parser_critique/byteman_rules/
```

### Inject malevolent data at runtime
As mentioned in [issue 10](https://github.com/dogukanucak/PA193_test_parser_SixEleven/issues/10), output_script_length is not
properly checked. We may demonstrate it with setting a malevolent value in the setter at runtime.
Byteman rule might look as follows, 
```
RULE inject crazy script_length value
CLASS SixElevenBlock
METHOD setOutput_script_length
AT ENTRY
IF true
DO traceln("KARM: Byteman, injecting output_script_length value -666...");
   $output_script_length = -666
ENDRULE
```
See [malevolent_injection.btm](./malevolent_injection.btm)

```
java -javaagent:${BYTEMAN_HOME}/lib/byteman.jar=script:${MY_BTM_RULES}/malevolent_injection.btm -jar target/pa193_test_parser_sixeleven.jar

Parsing the block...
KARM: Byteman, injecting output_script_length value -666...
Exception in thread "main" java.lang.StringIndexOutOfBoundsException: begin 304, end -1028, length 446
        at java.base/java.lang.String.checkBoundsBeginEnd(String.java:3116)
        at java.base/java.lang.String.substring(String.java:1885)
        at pa193.sixeleven.parser.SixElevenBlock.Parse_Hex(SixElevenBlock.java:295)
        at pa193.sixeleven.parser.Utils.displayBlock(Utils.java:114)
        at pa193.sixeleven.parser.PA193_test_parser_SixEleven.main(PA193_test_parser_SixEleven.java:31)
```

### Injecting into JDKs code
We claimed in [issue 8](https://github.com/dogukanucak/PA193_test_parser_SixEleven/issues/8) that the reg exp pattern is wastefully being compiled again and again and that it could be static. We prove there is no JDK's optimization coming to rescue and that it is really a bug.
```
RULE counting pattern compile calls
CLASS java.util.regex.Pattern
METHOD compile(String)
AT EXIT
BIND regex = $1
IF TRUE
DO traceln("Pattern", "Compiling pattern " + $1)
ENDRULE
```
See [counting_pattern_compile_calls.btm](./counting_pattern_compile_calls.btm)

We need to add ```boot``` and ```-Dorg.jboss.byteman.transform.all``` to be able to inject Byteman at JDK level.
```
java -javaagent:${BYTEMAN_HOME}/lib/byteman.jar=script:${MY_BTM_RULES}/counting_pattern_compile_calls.btm,boot:${BYTEMAN_HOME}/lib/byteman.jar -Dorg.jboss.byteman.transform.all -jar target/pa193_test_parser_sixeleven.jar
```
The trace log should contain the messages, let's see:
```
grep -c "Compiling pattern" trace000000000.log 
26
```
A fix could look like follows:

```
diff --git a/src/main/java/pa193/sixeleven/parser/SixElevenBlock.java b/src/main/java/pa193/sixeleven/parser/SixElevenBlock.java
index ab7a144..cb335bc 100644
--- a/src/main/java/pa193/sixeleven/parser/SixElevenBlock.java
+++ b/src/main/java/pa193/sixeleven/parser/SixElevenBlock.java
@@ -34,6 +34,10 @@ public class SixElevenBlock {
     private Utils utils = null;
     private boolean blockvalid = true;
 
+    private static final String pattern = "f9beb611[0-9a-fA-F]+0000";  // Checks if the block is in the correct format
+    // Create a Pattern object
+    private static final Pattern r = Pattern.compile(pattern);
+
     public SixElevenBlock(String blockhexstring) {
         this.blockhexstring = blockhexstring;
         blockvalid = hexValid(blockhexstring);
@@ -304,9 +308,6 @@ public class SixElevenBlock {
      * @return
      */
     private boolean hexValid(String hexstring) {
-        String pattern = "f9beb611[0-9a-fA-F]+0000";  // Checks if the block is in the correct format
-        // Create a Pattern object
-        Pattern r = Pattern.compile(pattern);
         // Now create matcher object.
         Matcher m = r.matcher(hexstring);
         return m.find();

```
We rebuild the program: ```mvn clean package```, remove the old log ```rm trace000000000.log```, run with Byteman again and we can see the pattern was compiled just once:
```
grep -c "Compiling pattern" trace000000000.log
1
```