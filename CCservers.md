Dev VM - Compute Canada VM - snapshot of production content (shared Fedora)
--

* dev-01.cwrc.ca (Nia dev Drupal instance) 
* dev-02.cwrc.ca (CWRC-Writer / OpenSky Drupal instance)
* dev-03.cwrc.ca (DE Drupal instance - to replace cwrc-dev-06.srv.ualberta.ca)

Notes re dev 01-03:
* June 2018: CWRC content as of Summer 2017 (last year). 
* Specific objects can easily be moved to this server 
* one day worth of downtime to update Fedora on the dev-01/02/03 instance

Test / Staging VM - Compute Canada VM - snapshot of production content (shared Fedora - intention to update snapshot at regular intervals)
--

* test-01.cwrc.ca (GitHub webhook automatically updates Drupal code with 'develop' branch)

* staging-01.cwrc.ca (GitHub webhook automatically updates Drupal code with 'master' branch)

Notes re test/staging: 
* 100% of production Fedora content as of June 5th, 2018. 
* The aim is to update test/staging with 100% of production content at regular intervals (time period to be determined)

Production - Compute Canada VM - production server
--
* prod-01.cwrc (to replace beta testing complete)
