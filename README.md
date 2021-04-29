# tech_documentation
Centralized storage for cwrc technical documentation. 

### CWRC-Writer

There are two main parts to a full CWRC-Writer installation, and each part runs more or less independently: 

CWRC-Writer - The editor itself that runs as javascript in the web browser.

CWRC-Server - the complementary backend services that run on a server and provide document storage, XML validation, and entity lookup.  These can be implemented however one would like - in any language or on any platform.

You might then ask:  ‘If the server can be implemented willy-nilly, how does the CWRC-Writer know how to make calls to the server to get documents, etc?”  Good question - the answer is that a bit more javascript code has to be written that knows how to interact with the given server.  The CWRC-Writer expects that this javascript will be written as node.js modules (CommonJS) and that each module will export a specific API to which the CWRC-Writer knows how to make calls (e.g., loadDocument()).  The server-specific modules take care of the plumbing: making calls to the server and setting returned data in the CWRC-Writer.  The modules also provide their own dialogs since any interaction with a given server is likely different.  We bundle these supporting modules together with the CWRC-Writer (which is itself setup as node.js module) using browserify, which creates a single javascript file that is then included in the index.html file.

Two CWRC-Writer installations that can be used as examples of how to put together a full CWRC-Writer installation are the [Islandora-CWRC-Writer](https://github.com/cwrc/Islandora-CWRC-Writer) and the [CWRC-GitWriter](https://github.com/cwrc/CWRC-GitWriter).

An overview of development practices for CWRC-Writer packages:

[CWRC-Writer-Dev-Docs](https://github.com/cwrc/CWRC-Writer-Dev-Docs/blob/master/README.md)

------------------

## Misc


### REST APIs

#### General: Islandora REST
CWRC offers a the Islandora REST as a means to interact programatically with repository. This section summerizes the more detailed documentation available here: https://github.com/discoverygarden/islandora_rest/blob/7.x/README.md

Definitions:
* PID: persistent identifier - FedoraCommons identifier for an object and part of the URI (commons.cwrc.ca/{PID} where {PID} is replaced with the object's PID

* DSID: DataStream ID or FedoraCommons datastream identifier - ID of the location where content is stored

##### General strategy (used outside of Drupal e.g., on another server)

* Step 1. Authentication against cwrc.ca (Islandora REST  Drupal Module). A command-line curl example that can be translated into programming language of choice.

```
curl -X POST -i -H "Content-type: application/json" -c token.txt -b token.txt -X POST https://${SERVER_NAME}/rest/user/login -d '{ "username":"${USERNAME}","password":"${PASSWORD}"}'
```

* Step 2. Lookup an object by PID (Persistent IDentifier)

```
curl -b token.txt -X GET https://${SERVER_NAME}/islandora/rest/v1/object/${PID}`
```

Notes:

* More details on the authentication and the basics to setup a session via cookies (only required if code is running outside of Drupal (e.g., microservice or batch job) and items are not publicly visable) are located in the following section of [Auth](https://github.com/cwrc/tech_documentation#authentication-against-apis)
* Server-side setup (not applicable for client access): an internal Google Doc including the above details and some repository side setup is included at the following link (but shouldn't be needed in this context)  [link](https://docs.google.com/document/d/1NBvM91g7XhUpens7e6UocaasZcaMhyt9ksYxRlb6ZWI/edit#heading=h.cixsnft4pbfa)

##### general usage of the REST API - common calls

```
given a {PID}

// lookup properties of the object via the REST endpoint
https://${SERVER_NAME}/islandora/rest/v1/object/{PID}

the JSON response contains properties 

// lookup content of a specified datastream via the REST endpoint
`https://${SERVER_NAME}/islandora/rest/v1/object/${PID}/datastream/${DSID}/?content=true`


Example REST calls:

// lookup properties of the object via the REST endpoint
`https://${SERVER_NAME}/islandora/rest/v1/object/orlando%3Ab4859cdd-8c58-46e9-bf2a-28bf8090fcbc`

// lookup content of a specified datastream via the REST endpoint
`https://${SERVER_NAME}/islandora/rest/v1/object/orlando%3Ab4859cdd-8c58-46e9-bf2a-28bf8090fcbc/datastream/CWRC/?content=true`

```

##### Downloading Content Pseudocode: given a collection PID, authenticate and download a specified datastream from all items in the collection

1. Authenticate: creates a token that is passed in as part of subsequent API requests. The user must have `view` access to the collection and all objects within the collection plus Drupal permissions to use the Islandora REST API (https://${SERVER_NAME}/admin/people/permissions): `View Objects` & `View Datastreams`

```
curl -X POST -i -H "Content-type: application/json" -c token.txt -b token.txt -X POST https://${SERVER_NAME}/rest/user/login -d '{ "username":"${USERNAME}","password":"${PASSWORD}"}'
```

2. define a set of objects (i.e., list of PIDs) to process, for example, lookup objects by ${COLLECTION_PID}. Note: a Solr query is an option to define the set objects; `rows` & `start` can be used for pagination of results plus the JSON response contains `numFound`. `RELS_EXT_isMemberOfCollection_uri_mt` allows defining the set by a CWRC collection. The `fl` parameter filters the response; remove to see the entire set of Solr fields. The parameter `sort=fgs_label_s+asc` will add a sort to the results. The `fgs_label_s` is a single valued Solr field (multivalued solr fields cannot with the `sort`). There is no Solr field for surname or name field that starts with surname -- one could be added.

```
curl -b token.txt -X GET "https://${SERVER_NAME}/islandora/rest/v1/solr/RELS_EXT_isMemberOfCollection_uri_mt:\"${COLLECTION_PID}\"?fl=PID&rows=999999&start=0&wt=json&sort=fgs_label_s+asc"
```

3. foreach object (i.e., PID) in step 2, retrieve the associated metadata

```
curl -b token.txt -X GET https://${SERVER_NAME}/islandora/rest/v1/object/${PID}
```

4. in the JSON metadata acquired during the previous step, use the `models` property to lookup the datastream ID (DSID) containing the XML -- below is the mapping

* if `cwrc:documentCModel` then DSID = `CWRC` // event/entry
* if `cwrc:citationCModel` then DSID = `MODS` // bibliography
* if `cwrc:person-entityCModel` then DSID = `PERSON` // person entity
* if `cwrc:organization-entityCModel` then DSID = `ORGANIZATION` // organization entity

5. request the contents (XML) within the specified datastream ${DSID} attached to the object ${PID}

Note: a download can occur whether or not you or someone else holds a lock on the object (see update content pseudocode for info on the locking mechanism).

```
curl -b token.txt -X GET https://${SERVER_NAME}/islandora/rest/v1/object/${PID}/datastream/${DSID}?content=true
```

More information on the REST API used above can be found here: https://github.com/discoverygarden/islandora_rest/blob/7.x/README.md

##### Updating Content Pseudocode: given an object PID, lock the object, download the specified datastream, process, and then upload the content back to CWRC

1. Determine how long you will need to process the objects and ask a CWRC admin to set the collection object locking time `/islandora/object/${COLLECTION_ID}/manage/collection ==> Manage lock objects`.

2. Follow the steps in the `Downloading content psuedocode` example above to gather the object contents you wish to process

3. Add to the above steps an API call to lock the object to prevent users from changing the item while you yourself are changing that item (overwriting others work since to original download) [details](https://github.com/echidnacorp/islandora_object_lock/blob/7.x/islandora_object_lock_rest/README.md)

    * Check if lock exists
        ```
        curl -b token.txt -X GET https://${SERVER_NAME}/islandora/rest/v1/object/${PID}/lock
        ```
    * Aquire lock
        ```
        curl -b token.txt -X POST https://${SERVER_NAME}/islandora/rest/v1/object/${PID}/lock
        ```

4. Process downloaded items

5. Once ready to place items back into the repository, update object datastream in repository (see notes below)

```
curl -b token.txt -X POST -F "method=PUT" -F "file=@${SOURCE_FILE}" https://${SERVER_NAME}/islandora/rest/v1/object/${PID}/datastream/${DSID}
```

6. Add workflow information describing the change [details here](https://github.com/cwrc/cwrc_workflow/blob/7.x/README.md) : ask a CWRC admin for the workflow parameters to add via the `activity` parameter

```
curl -b token.txt -G -X GET "https://${SERVER_NAME}/islandora_workflow_rest/v1/add_workflow" -d PID=${PID} -d activity='{"category":"metadata_contribution","stamp":"orlando:ENH","status":"c","note":"entity"}'
```

Note: documentation regarding the REST API update: "... mock PUT / DELETE requests as POST requests by adding an additional form-data field method to inform the server which method was actually intended.  At the moment multi-part PUT requests such as the one required to modify an existing datastream's content and properties are not implemented you can mock these PUT requests using aforementioned mechanism.POST and include an additional form-data field method with the value PUT...."

Note: other approaches to prevent write collisions:

* the metadata for a datastream (HTTP GET on the object DSID) contains a checksum. Saving the checksum at download time and then comparing to the checksum on the server at upload could act as a mechanism to verify the respository content has not been modified.

More information on the REST API used above can be found here: https://github.com/discoverygarden/islandora_rest/blob/7.x/README.md

##### Updating Content Pseudocode: given an object PID, lock the object, download the specified datastream, process, and then upload the content back to CWRC

1. Determine how long you will need to process the objects and ask a CWRC admin to set the collection object locking time `/islandora/object/${COLLECTION_ID}/manage/collection ==> Manage lock objects`.

2. Follow the steps in the `Downloading content psuedocode` example above to gather the object contents you wish to process

3. Add to the above steps an API call to lock the object to prevent users from changing the item while you yourself are changing that item (overwriting others work since to original download) [details](https://github.com/echidnacorp/islandora_object_lock/blob/7.x/islandora_object_lock_rest/README.md)

    * Check if lock exists
        ```
        curl -b token.txt -X GET https://${SERVER_NAME}/islandora/rest/v1/object/${PID}/lock
        ```
    * Aquire lock
        ```
        curl -b token.txt -X POST https://${SERVER_NAME}/islandora/rest/v1/object/${PID}/lock
        ```

4. Process downloaded items

5. Once ready to place items back into the repository, update object datastream in repository (see notes below)

```
curl -b token.txt -X POST -F "method=PUT" -F "file=@${SOURCE_FILE}" https://${SERVER_NAME}/islandora/rest/v1/object/${PID}/datastream/${DSID}
```

6. Add workflow information describing the change [details here](https://github.com/cwrc/cwrc_workflow/blob/7.x/README.md) : ask a CWRC admin for the workflow parameters to add via the `activity` parameter

```
curl -b token.txt -G -X GET "https://${SERVER_NAME}/islandora_workflow_rest/v1/add_workflow" -d PID=${PID} -d activity='{"category":"metadata_contribution","stamp":"orlando:ENH","status":"c","note":"entity"}'
```

7. Remove lock (if lock exists): `CWRC` datastream will not release the lock an update event like some other datastreams will (details: [exclusion list](https://github.com/cwrc/cwrc_islandora_tweaks/blob/develop/cwrc_islandora_tweaks.module#L190-L199), and CWRC-Writer `save` versus `save and exit` functionality.

```
curl -b token.txt -X DELETE https://${SERVER_NAME}/islandora/rest/v1/object/${PID}/lock
```


Note: documentation regarding the REST API update: "... mock PUT / DELETE requests as POST requests by adding an additional form-data field method to inform the server which method was actually intended.  At the moment multi-part PUT requests such as the one required to modify an existing datastream's content and properties are not implemented you can mock these PUT requests using aforementioned mechanism.POST and include an additional form-data field method with the value PUT...."

Note: other approaches to prevent write collisions:

* the metadata for a datastream (HTTP GET on the object DSID) contains a checksum. Saving the checksum at download time and then comparing to the checksum on the server at upload could act as a mechanism to verify the respository content has not been modified. Note all object have a checksum
```
curl -b token.txt -X GET https://${SERVER_NAME}/islandora/rest/v1/object/${PID}/datastream/${DSID}?content=false
```
* use the date on the created version
```
curl -b token.txt -X GET https://${SERVER_NAME}/islandora/rest/v1/object/${PID}/datastream/${DSID}?content=false
```

More information on the REST API used above can be found here: https://github.com/discoverygarden/islandora_rest/blob/7.x/README.md

#### Specific REST APIs: CWRC Workflow

How to access CWRC workflow information - <https://github.com/cwrc/cwrc_workflow>

#### Specific REST APIs: CWRC Entities

How to access CWRC entities - <https://github.com/cwrc/cwrc_entities>

#### Specific REST APIs: Object locking

How to lock and unlock and existing object - <https://github.com/echidnacorp/islandora_object_lock>, <https://github.com/echidnacorp/islandora_object_lock/blob/7.x/islandora_object_lock_rest/README.md>

#### Specific REST APIs: Credit Visualization

Lookup credit visualization details: <https://github.com/cwrc/islandora_cwrc_credit_visualization>

#### BagIT Extension

<https://github.com/cwrc/islandora_bagit_extension>

#### Preservation

To allow a preservation system to pull content from the CWRC repository, the following extenstion provides the required REST API's: [BagIt Extension](https://github.com/cwrc/islandora_bagit_extension).

The user needs to have view access to all Fedora objects. If the anonymous uses does not have access, then the REST Login API is required by the preservation client. 

#### Schema Validation

To allow the CWRC-Writer to validate a document against a schema via a web API (https://github.com/cwrc/cwrc-validator).

### Authentication against APIs

CWRC uses the Drupal "Services" and "Rest Server"

```
curl -X POST -i -H "Content-type: application/json" -c cookies.txt -b cookies.txt -X POST http://dev.local/rest/user/login -d '{ "username":"zz","password":"zz"}'
```

After which point you need only include “-b cookies.txt” for all subsequent requests for them to be authenticated as the zz user.

Like so:

```
curl -b cookies.txt -X GET http://dev.local/islandora/rest/v1/object/islandora:root
```

Assessing via jquery JavaScript:

```
Services -> Edit Resources -> select tab "Server" -> enable "application/x-www-form-urlencoded" to prevent " Unsupported request content type application/x-www-form-urlencoded"
http://stackoverflow.com/questions/8535820/drupal-login-via-rest-server
https://www.drupal.org/node/2279819
https://www.drupal.org/node/1334758
```

```
Javascript login - based on 2015-08-18 e-mail troubleshooting with Ed Armstrong
http://stackoverflow.com/questions/8863571/cors-request-why-are-the-cookies-not-sent
need to add xhrFields so cookies sent
may need but unsure as of 2015-08-18:
Header add Access-Control-Allow-Credentials "true"
Header add Access-Control-Allow-Methods: "GET, POST, PUT, DELETE"
Header add Access-Control-Allow-Headers: "Authorization"
```

```
    $.ajax({
        url: cwrcurl,
        type: 'GET',
        callback: '?',
        datatype: 'application/json',
        success: function() { alert("Success"); },
        error: function() { alert('Failed!'); },
        xhrFields: {
            withCredentials: true
        }
```

------------------

## CANARIE

ToDo: elaborate

Circa 2018-2021, [CANARIE](https://science.canarie.ca/researchsoftware/researchresource/main.html?resourceID=146) checks the health of the cwrc.ca site along with gathering usage stats plus housing links to information about the platform in CANARIE platform registry. The information provided to the CANARIE registry is provided by endpoints defined in [cwrc_core_tweaks](https://github.com/cwrc/cwrc_core_tweaks). URL's provided to the CANARIE registry either point to Drupal pages or to redirects such as the [Fact Sheet redirect](https://cwrc.ca/admin/config/search/redirect/edit/57?destination=admin/config/search/redirect/list/fact)

------------------

## CWRC Repository Drupal modules (all in Git format) as of 2017-07-18


#### Shared

* [all/libraries/CWRC-Dialogs](https://github.com/cwrc/CWRC-Dialogs)
* [all/libraries/CWRC-Mapping-Timelines-Project](https://github.com/cwrc/CWRC-Mapping-Timelines-Project)
* [all/libraries/CWRC-Writer](https://github.com/discoverygarden/CWRC-Writer)
* [all/libraries/tuque](https://github.com/Islandora/tuque)
* [all/modules/austese_collation](https://github.com:discoverygarden/austese_collation)
* [all/modules/austese_repository](https://github.com:discoverygarden/austese_repository)
* [all/modules/contrib/islandora_patches](islandora_patches)
* [all/modules/contrib/og](og)
* [all/modules/contrib/recaptcha/recaptcha-php](recaptcha-php)
* [all/modules/cwrc_islandora_xml](https://github.com/cwrc/cwrc_islandora_xml)
* [all/modules/cwrc_workflow](https://github.com/cwrc/cwrc_workflow)
* [all/modules/emicdora](https://github.com/discoverygarden/emicdora)
* [all/modules/emic_theme_feature](https://github.com/discoverygarden/emic_theme_feature)
* [all/modules/islandora](https://github.com/Islandora/islandora)
* [all/modules/islandora_bagit](https://github.com/Islandora/islandora_bagit)
* [all/modules/islandora_batch](https://github.com/discoverygarden/islandora_batch)
* [all/modules/islandora_binary_object](https://github.com/discoverygarden/islandora_binary_object)
* [all/modules/islandora_book_batch](https://github.com/discoverygarden/islandora_book_batch)
* [all/modules/islandora_bookmark](https://github.com/Islandora/islandora_bookmark)
* [all/modules/islandora_critical_edition](https://github.com/discoverygarden/islandora_critical_edition)
* [all/modules/islandora_critical_edition_advanced](https://github.com/discoverygarden/islandora_critical_edition_advanced)
* [all/modules/islandora_cwrc_writer](https://github.com/discoverygarden/islandora_cwrc_writer)
* [all/modules/islandora_embed-7.x-1.4-oulib](islandora_embed-7.x-1.4-oulib)
* [all/modules/islandora_find_replace](islandora_find_replace)
* [all/modules/islandora_fits](https://github.com/Islandora/islandora_fits)
* [all/modules/islandora_image_annotation](https://github.com/discoverygarden/islandora_image_annotation)
* [all/modules/islandora_importer](https://github.com/Islandora/islandora_importer)
* [all/modules/islandora_internet_archive_bookreader]( https://github.com/Islandora/islandora_internet_archive_bookreader)
* [all/modules/islandora_ip_embargo](https://github.com/Islandora/islandora_ip_embargo)
* [all/modules/islandora_jwplayer](https://github.com/Islandora/islandora_jwplayer/blob/mas://github.com/Islandora/ter/README.md)
* [all/modules/islandora_markup_editor](https://github.com/discoverygarden/islandora_markup_editor)
* [all/modules/islandora_oai](https://github.com/Islandora/islandora_oai)
* [all/modules/islandora_object_lock](https://github.com/echidnacorp/islandora_object_lock)
* [all/modules/islandora_ocr](https://github.com/Islandora/islandora_ocr)
* [all/modules/islandora_openseadragon](https://github.com/Islandora/islandora_openseadragon)
* [all/modules/islandora_paged_content](https://github.com/Islandora/islandora_paged_content)
* [all/modules/islandora_plupload](https://github.com/discoverygarden/islandora_plupload)
* [all/modules/islandora_pretty_text_diff](https://github.com/discoverygarden/islandora_pretty_text_diff)
* [all/modules/islandora_rest](https://github.com/discoverygarden/islandora_rest)
* [all/modules/islandora_scholar](https://github.com/discoverygarden/islandora_scholar)
* [all/modules/islandora_simple_workflow](https://github.com/Islandora/islandora_simple_workflow)
* [all/modules/islandora_solr_facet_pages](https://github.com/Islandora/islandora_solr_facet_pages)
* [all/modules/islandora_solr_metadata](https://github.com/Islandora/islandora_solr_metadata)
* [all/modules/islandora_solr_search](https://github.com/Islandora/islandora_solr_search)
* [all/modules/islandora_solr_views](https://github.com/Islandora/islandora_solr_views)
* [all/modules/islandora_solution_pack_audio](https://github.com/Islandora/islandora_solution_pack_audio)
* [all/modules/islandora_solution_pack_book](https://github.com/Islandora/islandora_solution_pack_book)
* [all/modules/islandora_solution_pack_collection](https://github.com/Islandora/islandora_solution_pack_collection)
* [all/modules/islandora_solution_pack_compound](https://github.com/Islandora/islandora_solution_pack_compound)
* [all/modules/islandora_solution_pack_entities](https://github.com/Islandora/islandora_solution_pack_entities)
* [all/modules/islandora_solution_pack_image](https://github.com/Islandora/islandora_solution_pack_image)
* [all/modules/islandora_solution_pack_large_image](https://github.com/Islandora/islandora_solution_pack_large_image)
* [all/modules/islandora_solution_pack_newspaper](https://github.com/Islandora/islandora_solution_pack_newspaper)
* [all/modules/islandora_solution_pack_pdf](https://github.com/Islandora/islandora_solution_pack_pdf)
* [all/modules/islandora_solution_pack_video](https://github.com/Islandora/islandora_solution_pack_video)
* [all/modules/islandora_user_entity_link](https://github.com/discoverygarden/islandora_user_entity_link)
* [all/modules/islandora_xacml_editor](https://github.com/Islandora/islandora_xacml_editor)
* [all/modules/islandora_xml_forms](https://github.com/Islandora/islandora_xml_forms)
* [all/modules/islandora_xquery](https://github.com/echidnacorp/islandora_xquery)
* [all/modules/job_scheduler/modules/job_scheduler_trigger](job_scheduler_trigger)
* [all/modules/objective_forms](https://github.com/Islandora/objective_forms)
* [all/modules/php_lib](https://github.com/Islandora/php_lib)
* [all/modules/pmgrowl????](https://www.drupal.org/project/pmgrowl)
* [all/modules/tei_content](https://www.drupal.org/project/1118172/git-instructions)
*
*
* [all/themes/bootstrap](bootstrap)
* [all/themes/contrib/aurora](aurora)
* [all/themes/custom/de_theme](de_theme)
* {all/themse/emic_theme](https://github.com/DGI-EMiC/emic-theme)
*
* [all/vendor/solarium/solarium]()
* [all/vendor/symfony/event-dispatcher]()


#### CWRC modules

* [default/libraries/CWRC-Mapping-Timelines-Project](https://github.com/cwrc/CWRC-Mapping-Timelines-Project)
* [default/libraries/CWRC-Writer](https://github.com/cwrc/CWRC-Writer)
* [default/libraries/basex-api - see islandora_cwrc_basexdb]()
* [default/libraries/ckeditor ???]()
* [default/libraries/islandora_cwrc_xslt_library ???](https://github.com/cwrc/islandora_cwrc_xslt_library)
*
* [default/modules/contrib/blockify](blockify)
* [default/modules/contrib/composer_manager](composer_manager)
* [default/modules/contrib/speedy](speedy)
*
* [default/modules/custom/block_islandora_options](https://github.com/echidnacorp/block_islandora_options)
* [default/modules/custom/borealis_block_wrappers](https://github.com/echidnacorp/borealis_block_wrappers)
* [default/modules/custom/cwrc_admin](https://bitbucket.org/digitalechidna/cwrc_admin)
* [default/modules/custom/cwrc_baseline](https://bitbucket.org/digitalechidna/cwrc_baseline)
* [default/modules/custom/cwrc_components](https://bitbucket.org/digitalechidna/cwrc_components)
* [default/modules/custom/cwrc_core_tweaks](https://github.com/cwrc/cwrc_core_tweaks)
* [default/modules/custom/cwrc_dashboards](https://bitbucket.org/digitalechidna/cwrc_dashboards)
* [default/modules/custom/cwrc_eap](https://github.com/cwrc/cwrc_eap)
* [default/modules/custom/cwrc_event](https://bitbucket.org/digitalechidna/cwrc_event)
* [default/modules/custom/cwrc_featured_projects](https://bitbucket.org/digitalechidna/cwrc_featured_projects)
* [default/modules/custom/cwrc_field_filler](https://github.com/cwrc/cwrc_field_filler)
* [default/modules/custom/cwrc_find_replace](https://github.com/cwrc/cwrc_find_replace)
* [default/modules/custom/cwrc_homepage_slider](https://bitbucket.org/digitalechidna/cwrc_homepage_slider)
* [default/modules/custom/cwrc_islandora_tweaks](https://github.com/cwrc/cwrc_islandora_tweaks)
* [default/modules/custom/cwrc_menu_links](https://bitbucket.org/digitalechidna/cwrc_menu_links)
* [default/modules/custom/cwrc_news](https://github.com/cwrc/cwrc_news)
* [default/modules/custom/cwrc_node_page](https://bitbucket.org/digitalechidna/cwrc_node_page)
* [default/modules/custom/cwrc_notification_rules](https://bitbucket.org/digitalechidna/cwrc_notification_rules)
* [default/modules/custom/cwrc_permissions](https://bitbucket.org/digitalechidna/cwrc_permissions)
* [default/modules/custom/cwrc_projects](https://bitbucket.org/digitalechidna/cwrc_projects)
* [default/modules/custom/cwrc_search](https://bitbucket.org/digitalechidna/cwrc_search)
* [default/modules/custom/cwrc_search_bar](https://github.com/cwrc/cwrc_search_bar)
* [default/modules/custom/cwrc_solr_site_content](https://bitbucket.org/digitalechidna/cwrc_solr_site_content)
* [default/modules/custom/cwrc_theme_compat](https://bitbucket.org/digitalechidna/cwrc_theme_compat)
* [default/modules/custom/cwrc_visualization](https://bitbucket.org/digitalechidna/cwrc_visualization)
* [default/modules/custom/cwrc_workflow_de](https://bitbucket.org/digitalechidna/cwrc_workflow_de)
* [default/modules/custom/cwrc_xacml](https://bitbucket.org/digitalechidna/cwrc_xacml)
* [default/modules/custom/de_contextual_help](https://bitbucket.org/digitalechidna/de_contextual_help)
* [default/modules/custom/islandora_blocks](https://github.com/echidnacorp/islandora_blocks)
* [default/modules/custom/islandora_object_field](https://bitbucket.org/digitalechidna/islandora_object_field)
* [default/modules/custom/islandora_saved_searches](https://github.com/echidnacorp/islandora_saved_searches)
* [default/modules/custom/og_global_roles](https://bitbucket.org/digitalechidna/og_global_roles)
* [default/modules/custom/translation_settings](https://bitbucket.org/digitalechidna/translation_settings)
* [default/modules/custom/webforms](https://bitbucket.org/digitalechidna/webforms)
* [default/modules/cwrc_entities](https://github.com/cwrc/cwrc_entities)
* [default/modules/cwrc_migration_batch](https://github.com/cwrc/cwrc_migration_batch)
* [default/modules/islandora_attach_datastream - is needed? If yes add git repo. Can it be replaced by islandora_xslt_paths?](https://github.com/Islandora/islandora_attach_datastream)
* [default/modules/islandora_collection_search](https://github.com/discoverygarden/islandora_collection_search)
* [default/modules/islandora_cwrc_basexdb](https://github.com/cwrc/islandora_cwrc_basexdb)
* [default/modules/islandora_cwrc_credit_visualization](https://github.com/cwrc/islandora_cwrc_credit_visualization)
* [default/modules/islandora_cwrc_document ???](https://github.com/cwrc/islandora_cwrc_document/islandora_cwrc_document)
* [default/modules/islandora_cwrc_writer ???](https://github.com/cwrc/islandora_cwrc_documentislandora_cwrc_writer)
* [default/modules/islandora_datastream_crud - is needed? If yes add git repo. Can it be replaced by islandora_xslt_paths?](https://github.com/Islandora/islandora_datastream_crud)
* [default/modules/islandora_plotit](https://github.com/cwrc/islandora_plotit)
* [default/modules/islandora_xslt_paths ???](https://github.com/cwrc/islandora_xslt_paths)

#### digitalpage.ca

* [digitalpage.ca/modules/islandora_digitalus-7.x-1.4](https://Semandra@bitbucket.org/Semandra/islandora-digitalus.git)

#### modernistcommons.ca

* [modernistcommons.ca/libraries/CWRC-Writer](https://github.com/discoverygarden/CWRC-Writer)
* [modernistcommons.ca/modules/agile_fonds_importer](https://github.com/agile-humanities/agile_fonds_importer)
* [modernistcommons.ca/modules/agile_customizations](https://github.com/agile-humanities/agile_yale_customizations)
* [modernistcommons.ca/modules/austese_collation](https://github.com/discoverygarden/austese_collation)
* [modernistcommons.ca/modules/austese_repository](https://github.com/discoverygarden/austese_repository)
* [modernistcommons.ca/modules/cwrc_workflow](https://github.com/discoverygarden/cwrc_workflow)
* [modernistcommons.ca/modules/emic_ctype_feature](https://github.com/discoverygarden/emic_ctype_feature)
* [modernistcommons.ca/modules/emic_theme_feature](https://github.com/discoverygarden/emic_theme_feature)
* [modernistcommons.ca/modules/emicdora](https://ackiejoe@github.com/discoverygarden/emicdora)
* [modernistcommons.ca/modules/islandora_batch](https://github.com/Islandora/islandora_batch)
* [modernistcommons.ca/modules/islandora_bookmark](https://github.com/Islandora/islandora_bookmark)
* [modernistcommons.ca/modules/islandora_book_batch](https://github.com/Islandora/islandora_book_batch)
* [modernistcommons.ca/modules/islandora_cwrc_writer](https://github.com/discoverygarden/islandora_cwrc_writer)
* [modernistcommons.ca/modules/islandora_feature_pack_ui](https://github.com/rosiel/islandora_feature_pack_ui)
* [modernistcommons.ca/modules/islandora_fits](https://github.com/Islandora/islandora_fits)
* [modernistcommons.ca/modules/islandora_image_annotation](https://github.com/Islandora/islandora_image_annotation)
* [modernistcommons.ca/modules/islandora_importer](https://github.com/Islandora/islandora_importer)
* [modernistcommons.ca/modules/islandora_internet_archive_bookreader](https://github.com/Islandora/islandora_internet_archive_bookreader)
* [modernistcommons.ca/modules/islandora_ip_embargo](https://github.com/Islandora/islandora_ip_embargo)
* [modernistcommons.ca/modules/islandora_jwplayer](https://github.com/Islandora/islandora_jwplayer)
* [modernistcommons.ca/modules/islandora_markup_editor](https://github.com/discoverygarden/islandora_markup_editor)
* [modernistcommons.ca/modules/islandora_oai](https://github.com/Islandora/islandora_oai)
* [modernistcommons.ca/modules/islandora_ocr](https://github.com/Islandora/islandora_ocr)
* [modernistcommons.ca/modules/islandora_openseadragon](https://github.com/Islandora/islandora_openseadragon)
* [modernistcommons.ca/modules/islandora_paged_content](https://github.com/Islandora/islandora_paged_content)
* [modernistcommons.ca/modules/islandora_plupload](https://github.com/discoverygarden/islandora_plupload)
* [modernistcommons.ca/modules/islandora_rest](https://github.com/discoverygarden/islandora_rest)
* [modernistcommons.ca/modules/islandora_scholar](https://github.com/Islandora/islandora_scholar)
* [modernistcommons.ca/modules/islandora_simple_workflow](https://github.com/Islandora/islandora_simple_workflow)
* [modernistcommons.ca/modules/islandora_solr_metadata](islandora_solr_metadata)
* [modernistcommons.ca/modules/islandora_solr_search](islandora_solr_search)
* [modernistcommons.ca/modules/islandora_solr_views](islandora_solr_views)
* [modernistcommons.ca/modules/islandora_solution_pack_audio](https://github.com/Islandora/islandora_solution_pack_audio)
* [modernistcommons.ca/modules/islandora_solution_pack_book](https://github.com/Islandora/islandora_solution_pack_book)
* [modernistcommons.ca/modules/islandora_solution_pack_collection](https://github.com/Islandora/islandora_solution_pack_collection)
* [modernistcommons.ca/modules/islandora_solution_pack_compound](islandora_solution_pack_compound)
* [modernistcommons.ca/modules/islandora_solution_pack_entities](https://github.com/Islandora/islandora_solution_pack_entities)
* [modernistcommons.ca/modules/islandora_solution_pack_html_snippet](https://github.com/agile-humanities/islandora_solution_pack_html_snippet)
* [modernistcommons.ca/modules/islandora_solution_pack_image](https://github.com/agile-humanities/islandora_solution_pack_image)
* [modernistcommons.ca/modules/islandora_solution_pack_large_image](https://github.com/agile-humanities/islandora_solution_pack_large_image)
* [modernistcommons.ca/modules/islandora_solution_pack_newspaper](https://github.com/agile-humanities/islandora_solution_pack_newspaper)
* [modernistcommons.ca/modules/islandora_solution_pack_pdf](https://github.com/agile-humanities/islandora_solution_pack_pdf)
* [modernistcommons.ca/modules/islandora_solution_pack_video](https://github.com/agile-humanities/islandora_solution_pack_video)
* [modernistcommons.ca/modules/islandora_xacml_editor](https://github.com/agile-humanities/islandora_xacml_editor)
* [modernistcommons.ca/themes/emic-theme](https://github.com/discoverygarden/emic-theme)

#### spanishcivilwar.ca

* [spanishcivilwar.ca/themes/mounta-civil-theme](https://github.com/echidnacorp/mounta-civil-theme)



## Odds and ends
### Handy commands

To find readme files
find . -name README.md  > /tmp/z
sed 's%/[^/]*$%%' /tmp/z
sed 's%/\([^/]*\)$%/\1](\1)%' /tmp/z > /tmp/zzzzz
grep -R 'url = ' */.git/config

