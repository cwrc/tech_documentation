# CWRC platform components:

Please refer to the [Complete list of CWRC components](https://github.com/cwrc/tech_documentation/blob/master/README.md#cwrc-repository-drupal-modules-all-in-git-format-as-of-2017-07-18) for the provenance of each CWRC component.

# Validation procedures

## Newly-added third-party components:

1. New proposed code is reviewed by the CWRC technical lead and Project Manager
2. Once recommended by the CWRC technical lead and project manager, the addition is veted by the CWRC Project Lead
3. The new code is added to the CWRC test server where it undergoes extensive testing to root out any incompatibility issues with the rest of the platform and to assess the functionality as described in the new software documentation
4. Once this review is complete, the software is pushed to the staging server.

## Updated third-party components:

1. Updated code is reviewed by the CWRC technical lead and Project Manager
2. Once recommended by the CWRC technical lead and project manager, the update is veted by the CWRC Project Lead
3. The update is added to the CWRC test server where it undergoes extensive testing to root out any incompatibility issues with the rest of the platform and to assess any newlly added functionality
4. Once this review is complete, the update is pushed to the staging server.


## Newly added in-house components:

1. Any new in-house component is developed on one of the CWRC development enviornments
2. Once developemtn is compplete, the code is QAed locally by the CWRC project manager and submitted for review to the Project Lead
3. Once the new software is vetted by the Project Lead, it is added to the CWRC git architecture and pushed to the test server where it is QAed again to root out any incompatibility issues with the rest of the platform
4. Once this review is complete, the update is pushed to the staging server.

## Updated in-house components:

1. Any update to the in-house components is developed on one of the CWRC development enviornments
2. Once developement is compplete, the code is QAed locally by the CWRC project manager and submitted for review to the Project Lead
3. Once the new software is vetted by the Project Lead, it is added to the CWRC git architecture and pushed to the test server where it is QAed again to root out any incompatibility issues with the rest of the platform
4. Once this review is complete, the update is pushed to the staging server.

## CWRC releases:

### "Regular" releases:

Based on the amount of new or updated software sitting on the staging server, CWRC releases are scheduled at the discretion of the CWRC Project Leader, after consulting with the rest of the CWRC team (project manager, technical lead, developers).
The actual release is preceded by another round of testing on the staging server and is documented at https://github.com/cwrc/beta.cwrc.ca/releases.


### Hotfixes, security patches:

As time is of the essence for this type of software release, we are installing them on the CWRC platform as soon asthey become available. 
·         
