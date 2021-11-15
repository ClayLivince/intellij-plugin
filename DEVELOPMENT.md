# IntelliJ plugin for Python support

## Summary for the JetBrains Marketplace
- File `plugin-description.md` is the description shown on the Marketplace. This is markdown, which is converted to HTML, before the plugin is published. Edit this file to update the description shown on plugins.jetbrains.com.
- File `plugin-changelog.md` is the changelog. The first section is automatically added to the plugin when it's published.

## Summary for developers

## Build
Gradle is used as build system. The gradle plugin https://github.com/JetBrains/gradle-intellij-plugin automatically downloads the IntelliJ files to build against.

- Run all tests: `./gradlew clean test`
- Build the obfuscated plugin zip file: 
`bash build-plugin.bash` or `./gradlew clean build && (cd proguard && bash run-proguard.bash)`

### Executing
IntelliJ and PyCharm require **version 8.x of the Java SDK**. Both OpenJDK and Oracle JDK are compatible.

The plugin can be executed within IntelliJ:
    ```./gradlew runIde```

#### Executing with specific versions of an IDE
Please refer the section for testers above.

### Building the plugin
 ```./gradlew clean build```
The plugin zip file is saved in build/distribution/
If you'd like to skip the tests, then run:
 ```./gradlew clean build -x test```

## Implementation details

### HTTP processing
All http calls are executed in a background thread. There is only one
connection to the kite backend to make sure that events and calls arrive in
the desired order and to make sure that kite is not overloaded with too many requests.

### Event processing
Editor events are queued. Events of the same actionType for the same file are bundled,
e.g. only one edit event is sent if two consecutive edit events are placed into the queue.
The special case of an edit event followed by a selection event is replaced with an edit event
which has the state (buffer content + cursor position) of the last event, i.e. the selection event.

The queued items are processed after a fixed delay (currently 250ms). The items are processed oldest to newest
and send to the kite http backend.

### Code completion
Code completion is only available in supported file types.

A code completion call is executed as soon as the event queue is empty to make
sure that all events have been processed. If there are code completions then any following code completion suggestions by PyCharm or other plugins are supressed.

If kite returned no completions or if a timeout (currently 350ms) occurred while waiting for an empty event queue then
Kite's completions are skipped. The default code completion contributors are called in every case, i.e. a fallback to other code completion suggestions is always done.

### Whitelisting
If a user works in a file which is not yet in a whitelisted directory then IntelliJ will display a notification to ask whether the user would like to
add it to the list of whitelisted resources.
The exact workflow is defined in Kite's developer configuration.
The notifications displayed are non-model balloon popups. This kind of background notification is used to be as unobtrusive as possible.
A modal dialog would steal the focus and would possibly annoy users which are interruped in their work.
The notification offers these options:
 - Whitelist /dir/send/by/kite
 - Browse ...
 - Settings ...
 - Ignore this file

## Development
### Logging
If you would like to see detailed log message you have to enable them in PyCharm. Do this to enable debug messages:

    - Open `Help -> Debug Log Settings` PyCharm
    - Enter `kite` in a new line in the text editor
    - Restart PyCharm

The log data is available in the file opened by `Help -> Show Log in Finder`

If the trace level of the logger-category kite.http is enabled then the log will contain the curl command lines to re-execute a kite request to simplify debugging.
The trace level can be activated by entering `kite.http` as a new line in `Help -> Debug Log Settings`.

#### Available channels
These channels can be used in the `Debug Log Settings` dialog to get more verbose plugin log output:
- #kite.action
- #kite.action.delegate
- #kite.action.delegateHandler
- #kite.action.override
- #kite.api
- #kite.completions
- #kite.eventQueue
- #kite.http
- #kite.http.script (trace only)
- #kite.link.externalExample
- #kite.link.linkListener
- #kite.link.member
- #kite.mockHttp
- #kite.monitoring
- #kite.monitoring.preload
- #kite.pebble
- #kite.pebble.extension
- #kite.platform
- #kite.platform.exeDetector
- #kite.platform.macExeDetector
- #kite.platform.pathLauncher
- #kite.renderUtil
- #kite.rollbar
- #kite.status.model
- #kite.status.statusBarWidget
- #kite.timing
- #kite.ui
- #kite.ui.SwingWorker
- #kite.ui.autoPopup
- #kite.ui.docPopupManager
- #kite.ui.mouseOverManager
- #kite.ui.quickDocProxy
- #kite.ui.quickDocSupression
- #kite.ui.signaturePopup
- #kite.uil.htmlPopup
- #kite.whitelisting
- #kite.whitelisting.link

### Templates
Kite responses are rendered using the Pebble template engine (http://www.mitchellbosecke.com/pebble/home).

#### Template development
It's possible to let the plugin use template files available on disk instead of the built-in data.
If $HOME/intellij-templates/ is available, then this directory will be used as base path to load the templates. The directory should contain the data of https://github.com/kiteco/intellij-plugin-private/tree/feature/documentationLookup/src/main/resources/templates/ .

It's also possible to define the Java property *kite.templatePath* to define where the template directory is available on disk.
To define it the custom VM options of PyCharm must be edited:

- Open PyCharm
- Choose `Help > Edit Custom VM Options ...`
- Agree to create the file, if it does not exist yet
- Add a new line at the end of the file (adjusted to your configuration):

        -Dkite.templatePath=/Users/username/git/intellij-plugin-private/src/main/resources/templates

- Restart PyCharm

Changes to the template files will be loaded as soon as a new lookup is requested.

If the property *kite.templateOutputPath.hover* is defined, then the last rendering of the hover report will be written into a file at the given location.
For example, if the line

    -Dkite.templateOutputPath.hover=/Users/username/intellij-html-output.html

is given in the configuration file above, then each invocation of the hover report will update the content of the file */Users/username/intellij-html-output.html* with the html data displayed in the popup window shown in IntelliJ.

#### Template directory structure
The templates are for Python only, at the moment. Each supported language will habe its own set of rendering templates.

- The shared template pieces are in `templates/python/common/`
- Shared macros are defined in `templates/python/common/macros.peb`
- The hover report is rendered by `templates/hover/long/hover.peb`.
- A symbol report, i.e. a reponse of `/api/editor/symbol/ID`, is rendered by `templates/symbolReport/symbolReport.peb`.
- A value report, i.e. a reponse of `/api/editor/value/ID`, is rendered by templates/valueReport/valueReport.peb.
- Signature information is rendered by `templates/python/signature/calls.peb` and `templates/python/signature/call.peb`

Each template file has a header which lists the context variabels which are available. The given types refer to Java classes available in `src/main/java/com/kite/intellij/backend`.

Include directives in the template refer to the path starting at `src/main/resources/templates`, i.e. base.peb is references as `common/base.peb`.

### Properties defined for all templates
The following properties are always available in a template:
- `dark`: actionType `boolean`, is true if the current theme is Darcula, i.e. dark colors
- `light`: actionType `boolean`, is true if the current theme is the default theme, i.e. lighter colors. `light` is always the inverted value of `dark`
- `windows`: actionType `boolean`, is true if the current OS is any version of Windows
- `mac`: actionType `boolean`, is true if the current OS is any version of Mac OS X
- `linux`: actionType `boolean`, is true if the current OS is any version of Linux
- `bgColor`: actionType `String`, is the hex-encoded value (including the prefix `#`) of the currently configured background color for IntelliJ's documentation panel
- `textColor`: actionType `String`, is the hex-encoded value (including the prefix `#`) of the currently configured text color


#### Pebble extensions
The kite plugin adds several extensions to the Pebble rendering.

##### Functions
The following custom pebble functions are available in addition to the built-in functions:

- `kiteInviteLink`: This function returns a link which redirects to the "Invite" page on kite.com. The link needs to opened in the browser because it redirects.

##### Filters
The following filters are made available in addition to the built-in filters:

- `kiteExternalLink`: This filter takes a value of tyep `string`, `Id`, `Value`, `ValueExt`, `Symbol` or `SymbolExt` and returns an URL which references the external documentation rendering as provided by kite.
- `kiteInternalLink`: This filter takes a value of actionType `string`, `Id`, `Value`, `ValueExt`, `Symbol` or `SymbolExt` and returns an URL which linkes to the built-in view to show the details.
- `kiteLinksLink`: This filter takes a value of actionType `string`, `Id`, `Value`, `ValueExt` and returns an URL which will open the list of links in the built-in view.
- `kiteExample`: returns an url which will open the example in the web, e.g. `{{ example.id | kiteExample }}`
- `kiteFilename`: returns just the filename of an input path, e.g. `{{ fullPath | kiteFilename }}`
- `kiteSignatureInfoLink`: named parameters `expandCommonInvocations` and `argIndex`. Returns an URL which links to a signature information for the element at the current editor's cursor position.
- `kiteCopilotDocsLink`: generates an URL which opens the input symbol in the Copilot
- `kiteClasspathUrl`: generates an URL which points to a resource in the classpath. This can be used to reference images bundled with the plugin, e.g. `{{ "/icons/logo.png" | kiteClasspathUrl }}` can be used as a value for an image src attribute.
- `kiteColor`: named parameters `brighter` and `darker` which take tone levels, e.g. `{{ "#ffffff" | kiteColor("darker"=3) }}` will reduce the brightness of white by 3 tones
- `kiteFileUrl`: named parameters `line`. Takes a string as input and returns an URL referencing it if it exists on the local file system. If it doesn't exist then null is returned. The parameter `line` is added query parameter, if defined.

# Used libraries and licenses
This is an overview of the dependencies and their licenses.

- [flying saucer](https://github.com/flyingsaucerproject/flyingsaucer/), XHTML/CSS 2.1 rendering library, [LGPL 2.1 or later](https://github.com/flyingsaucerproject/flyingsaucer/blob/master/LICENSE)
- [JSoup](https://jsoup.org/), HTML parsing and cleaning, [MIT](https://jsoup.org/license)
- [Pebble](https://github.com/PebbleTemplates/pebble), template rendering engine, [BSD-3-clause](https://github.com/PebbleTemplates/pebble/blob/master/LICENSE)
- [Rollbar](https://github.com/rollbar/rollbar-java), Rollbar integration library, [MIT](https://github.com/rollbar/rollbar-java/blob/master/LICENSE)
- [coverity-escapers](https://github.com/coverity/coverity-security-library), text escaping library used by Pebble, [Custom coverity license](https://github.com/coverity/coverity-security-library#license)
- [Jetty HTTP client](https://www.eclipse.org/jetty/), HTTP client, [Apache 2 and Eclipse dual license](https://github.com/eclipse/jetty.project/blob/jetty-9.4.x/LICENSE-eplv10-aslv20.html)
- [NanoHttpd](https://github.com/NanoHttpd/nanohttpd), HTTP server for unit testing, [BSD-3-clause](https://github.com/NanoHttpd/nanohttpd/blob/master/LICENSE.md)
- [backo-java](https://github.com/segmentio/backo-java), [MIT](https://github.com/segmentio/backo-java/blob/master/LICENSE.md)
- [okhttp](https://github.com/square/okhttp), [Apache 2.0](https://github.com/square/okhttp/blob/master/LICENSE.txt)
- [okio](https://github.com/square/okio), [Apache 2.0](https://github.com/square/okio/blob/master/LICENSE.txt)
- [analytics](https://github.com/segmentio/analytics-java), [MIT](https://github.com/segmentio/analytics-java)
- [retrofit](https://github.com/square/retrofit), [Apache 2.0](https://github.com/square/retrofit/blob/master/LICENSE.txt)
