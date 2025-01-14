---
title: "app"
linkTitle: "app"
weight: 5
description: >
  **App Forms**: Used to complete reports, tasks, and actions in the app
relevantLinks: >
  docs/apps/concepts/workflows
  docs/design/best-practices
keywords: workflows app-forms
---

App forms are used for care guides within the web app, whether accessed in browser or via the Android app. When a user completes an app form, the contents are saved in the database with the type `data_record`. These docs are known in the app as [Reports]( {{< ref "apps/features/reports" >}} ).

App forms are defined by the following files:

- A XML form definition using a CHT-enhanced ODK XForm format
- A XLSForm form definition, easier to write and converts to the XForm (optional)
- Meta information in the `{form_name}.properties.json` file (optional)
- Media files in the `{form_name}-media` directory (optional)

## XForm

A CHT-enhanced version of the ODK XForm standard is supported.

Data needed during the completion of the form (eg patient's name, prior information) is passed into the `inputs` group. Reports that have at least one of `place_id`, `patient_id`, and `patient_uuid` at the top level will be associated with that contact. 

{{< see-also page="apps/reference/forms/contact" title="Passing contact data to care guides" >}}

A typical form ends with a summary group (eg `group_summary`, or `group_review`) where important information is shown to the user before they submit the form.

In between the `inputs` and the closing group is the form flow - a collection of questions that can be grouped into pages. All data fields submitted with a form are stored, but often important information that will need to be accessed from the form is brought to the top level. To make sure forms are properly associated to a contact, make sure at least one of `place_id`, `patient_id`, and `patient_uuid` is stored at the top level of the form.

{{< see-also page="design/best-practices" anchor="content-and-layout" title="Content and Layout" >}}

## XLSForm

Since writing raw XML can be tedious, we suggest creating the forms using the [XLSForm standard](http://xlsform.org/), and using the [medic-conf](https://github.com/medic/medic-conf) command line configurer tool to [convert them to XForm format](#build).

| type | name | label | relevant | appearance | calculate | ... |
|---|---|---|---|---|---|---|
| begin group | inputs | Inputs | ./source = 'user' | field-list |
| hidden | source |
| hidden | source_id |
| begin group | contact |
| db:person | _id | Patient ID |  | db-object |
| string | patient_id | Medic ID |  | hidden |
| string | name | Patient Name |  | hidden |
| end group
| end group
| calculate | _id | | | | ../inputs/contact/_id |
| calculate | patient_id | | | | ../inputs/contact/patient_id |
| calculate | name | | | | ../inputs/contact/name |
| ...
| begin group | group_summary | Summary |  | field-list summary |
| note | r_patient_info | \*\*${patient_name}\*\* ID: ${r_patient_id} |
| note | r_followup | Follow Up \<i class="fa fa-flag"\>\</i\> |
| note | r_followup_note | ${r_followup_instructions} |
| end group |

### Supported XLSForm Meta Fields
[XLSForm](http://xlsform.org/) has a number of [data type options](https://xlsform.org/en/#metadata) available for meta data collection, of which the following are supported in CHT app forms:

| element | description |
|---|---|
| `start` | A timestamp of when the form entry was started, which occurs when the form is fully loaded. |
| `end` | A timestamp of when the form entry ended, which is when the user hits the Submit button. |
| `today` | Day on which the form entry was started. |

## XPath
Calculations are achieved within app forms using XPath statements in the "calculate" field of XForms and XLSForms. CHT apps support XPath from the [ODK XForm spec](https://getodk.github.io/xforms-spec), which is based on a subset of [XPath 1.0](https://www.w3.org/TR/1999/REC-xpath-19991116/), and is evaluated by [`medic/openrosa-xpath-evaluator`](https://github.com/medic/openrosa-xpath-evaluator). The ODK XForm documentation provides useful notes about the available [operators](https://getodk.github.io/xforms-spec/#xpath-operators) and [functions](https://getodk.github.io/xforms-spec/#xpath-functions). Additionally, [CHT specific functions](#cht-xpath-functions) are available for forms in CHT apps.

{{% alert title="Note" %}} 
The `+` operator for string concatenation is deprecated and will be removed in a future version. You are strongly encouraged to use the [`concat()`](https://getodk.github.io/xforms-spec/#fn:concat) function instead. 
{{% /alert %}}



## CHT XForm Widgets

Some XForm widgets have been added or modified for use in the app:
- **Bikram Sambat Datepicker**: Calendar widget using Bikram Sambat calendar. Used by default for appropriate languages.
- **Countdown Timer**: A visual timer widget that starts when tapped/clicked, and has an audible alert when done. To use it create a `note` field with an `appearance` set to `countdown-timer`. The duration of the timer is the field's value, which can be set in the XLSForm's `default` column. If this value is not set, the timer will be set to 60 seconds.
- **Contact Selector**: Select a contact, such as a person or place, and save their UUID in the report. In v3.10.0 or above, set the field type to `string` and appearance to `select-contact type-{{contact_type_1}} type-{{contact_type_2}} ...`. If no contact type appearance is specified then all contact types will be returned. For v3.9.0 and below, set the field type to `db:{{contact_type}}` and appearance to `db-object`.
- **Rapid Diagnostic Test capture**: Take a picture of a Rapid Diagnotistic Test and save it with the report. Works with [rdt-capture Android application](https://github.com/medic/rdt-capture/). To use create a string field with appearance `mrdt-verify`.
- **Simprints registration**: Register a patient with the Simprints biometric tool. To include in a form create a `string` field with `appearance` of `simprints-reg`. Requires the Simprints app connected with hardware, or [mock app](https://github.com/medic/mocksimprints). Demo only, not ready for production since API key is hardcoded.

The code for these widgets can be found in the [Medic repo](https://github.com/medic/medic/tree/master/webapp/src/js/enketo/widgets).

### Contact Selector

Using a contact selector allows you to get data off the selected contact(person or place) or search for an existing contact. 

When using with the appearance column set to `db-object`. The contact selector will display as a search box allowing you to search for the type of contact specified when building the report. EX: `db-person` will only search for contacts with type of person. 

When used as a field you can pull the current contact. This is can be used to link reports to a person or place where you started the form from. Getting the data of `_id` or `patient_id` and setting those to `patient_id` or `patient_uuid` on the final report will link that report so it displays on their contact summary page. 

Example of getting the data from the contact and assigning it to the fields neccessary to link the report. 

| type | name | label | relevant | appearance | calculate | ... |
|---|---|---|---|---|---|---|
| begin group | contact |
| db:person | _id | Patient ID |  | db-object |
| string | patient_id | Medic ID |  | hidden |
| string | name | Patient Name |  | hidden |
| end group
| calculate | patient_uuid | Patient UUID| ||../contact/_id|
| calculate | patient_id | Patient ID| ||../contact/patient_id|
 
## CHT XPath Functions

### `difference-in-months`

Calculates the number of whole calendar months between two dates. This is often used when determining a child's age for immunizations or assessments.

### `z-score`

In Enketo forms you have access to an XPath function to calculate the z-score value for a patient. The function accesses table data stored in CouchDB.

The `z-score` function takes four parameters: 
- The name of z-score table to use, which corresponds to value of the database document's `_id` attribute.
- Patient's sex, which corresponds to the data object's name. In the example below `male` for this parameter corresponds to `charts[].data.male` in the database document.
- First parameter for the table lookup, such as age. Value maps to the `key` value in the database document.
- Second parameter for the table lookup, such as height. Value is compared against the `points` in the databae document.

#### Example Use
[This example XForm form](https://github.com/medic/cht-core/blob/master/demo-forms/z-score.xml) shows the use of the z-score function. To calculate the z-score for a patient given their sex, age, and weight the XPath calculation is as follows:

`z-score('weight-for-age', ../my_sex, ../my_age, ../my_weight)`

The data used by this function needs to be added to CouchDB. The example below shows the structure of the database document. It creates a `weight-for-age` table, where you can see that a male aged 0 at 2.08kg has a z-score of -3. Your database doc will be substantially larger, so you may find the [conversion script](https://github.com/medic/cht-core/blob/master/scripts/zscore-table-to-json.js) helpful to convert z-score tables to the required doc format.


```json
{
  "_id": "zscore-charts",
  "charts": [
    {
      "id": "weight-for-age",
      "data": {
        "male": [
          {
            "key": 0,
            "points": [
              1.701,
              2.08,
              2.459,
              2.881,
              3.346,
              3.859,
              4.419,
              5.031,
              5.642
            ]
          }
        ]
      }
    }
  ]
}
```

## CHT Special Fields

The `NationalQuintile` and `UrbanQuintile` fields on a form will be assigned to all people belonging to the place. This is helpful when household surveys have quintile information which could be used to target health services for individuals. {{< see-also prefix="Read More" page="apps/guides/forms/wealth-quintiles" >}} 

## Uploading Binary Attachments

Forms can include arbitrary binary data which is submitted and included in the doc as an attachment. If this is an image type it'll then be displayed inline in the report UI.

To mark an element as having binary data add an extra column in the XLSForm called `instance::type` and specify `binary` in the element's row.

## Properties

The meta information in the `{form_name}.properties.json` file defines the form's title and icon, as well as when and where the form should be available.

### `forms/app/{form_name}.properties.json`

| property | description | required |
|---|---|---|
| `title`| The form's title seen in the app. Uses a localization array using the 2-letter code, not the translation keys discussed in the [Localization section]({{< relref "apps/reference/translations" >}}). | yes |
| `icon` | Icon associated with the form. The value is the image's key in the `resources.json` file, as described in the [Icons section]({{< relref "apps/reference/resources#icons" >}}) | yes |
| `subject_key` | Override the default report list title with a custom translation key. The translation uses a summary of the report as the evaluation context so you can include report fields in your value, for example: `Case registration {{case_id}}`. Useful properties available in the summary include: `from` (the phone number of the sender), `phone` (the phone number of the report contact), `form` (the form code), `subject.name` (the name of the subject), and `case_id` (the generated case id). Defaults to the name of the report subject. | no |
| `hidden_fields` | Array of Strings of form fields to hide in the view report UI in the app. This is only applied to future reports and will not change how existing reports are displayed. | no |
| `context` | The context defines when and where the form should be available in the app | no |
| `context.person` | Boolean determining if the form can be seen in the Action list for a person's profile. This is still subject to the `expression`. | no |
| `context.place` | Boolean determining if the form can be seen in the Action list for a person's profile. This is still subject to the `expression`. | no |
| `context.expression` | A JavaScript expression which is evaluated when a contact profile or the reports tab is viewed. If the expression evaluates to true, the form will be listed as an available action. The inputs `contact`, `user`, and `summary` are available. By default, forms are not shown on the reports tab, use `"expression": "!contact"` to show the form on the Reports tab since there is no contact for this scenario. | no |
| `context.permission` | String permission key required to allow the user to view and submit this form. If blank, this defaults to allowing all access. | no |

### Code sample

In this sample properties file, the associated form would only show on a person's page, and only if their sex is unspecified or female and they are between 10 and 65 years old:

#### `forms/app/pregnancy.properties.json`
```json
  {
    "title": [
      {
        "locale": "en",
        "content": "New Pregnancy"
      },
      {
        "locale": "hi",
        "content": "नई गर्भावस्था"
      }
    ],
    "icon": "pregnancy-1",
    "hidden_fields": [ "private", "internal" ],
    "context": {
      "person": true,
      "place": false,
      "expression": "contact.type === 'person' && (!contact.sex || contact.sex === 'female') && (!contact.date_of_birth || (ageInYears(contact) >= 10 && ageInYears(contact) < 65))",
      "permission": "can_register_pregnancies"
    }
  }
```

## Build
    
Convert and build the app forms into your application using the `convert-app-forms` and `upload-app-forms` actions in `medic-conf`.

    medic-conf --local convert-app-forms upload-app-forms
