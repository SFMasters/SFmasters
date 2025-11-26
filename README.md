.review-container {
    border: 1px solid #ccc;
    border-radius: 5px;
    padding: 1rem 1.5rem;
    background: #fff;
    margin-top: 1rem;
}

.review-title {
    font-weight: bold;
    margin-bottom: 0.5rem;
    font-size: 1.1rem;
}

.object-label {
    font-weight: bold;
    color: #275bb9;
    margin-bottom: 0.7rem;
    display: block;
}

.field-list {
    margin-bottom: 1rem;
    margin-top: 0;
}

.field-row {
    margin-bottom: 0.45rem;
}

.field-label {
    font-weight: bold;
    margin-right: 0.3rem;
}

.field-separator {
    margin-right: 0.25rem;
    color: #666;
}

.field-value {
    color: #222;
    white-space: pre-wrap;
    word-break: break-word;
}

.button-area {
    margin-top: 1rem;
}

.no-recommendations-container {
    background-color: #f3f3f3;
    border: 1px solid #d8d8d8;
    border-radius: 4px;
    padding: 1.5rem;
    margin-top: 1rem;
    text-align: left;
}

.no-recommendations-title {
    font-size: 1.125rem;
    font-weight: 600;
    color: #3e3e3c;
    margin-bottom: 0.75rem;
}

.no-recommendations-message {
    font-size: 0.875rem;
    color: #706e6b;
    line-height: 1.5;
}

.card-title {
    font-family: "Salesforce Sans", Arial, sans-serif;
    font-size: 13px;
    font-weight: 700;
    color: #181818;
    margin-bottom: 0.75rem;
    margin-top: 0.5rem;
}

.review-info-msg {
    font-family: "Salesforce Sans", Arial, sans-serif;
    font-size: 13px;
    font-weight: 400;
    color: #1b263c; /* Salesforce SLDS Core Text Default */
    margin-bottom: 1rem;
}

.review-container.slds-box {
    margin-bottom: 1rem;
    background: #fff;
    border: 1px solid #d8dde6;
    border-radius: 0.25rem;
    padding: 1rem 1.5rem 1rem 1.5rem;
    position: relative;
}

.close-btn {
    position: absolute;
    top: 0.5rem;
    right: 0.5rem;
    z-index: 1;
}

.button-area {
    margin-bottom: 0.5rem;
    margin-top: 0;
}

.regen-btn {
    margin-top: 1rem;
}

/* Task Details Styling - Enhancement for displaying task fields in follow-up actions */
/* Added to support inline display of Subject, Due Date, Assigned To, Related To fields */

.task-details {
    background-color: transparent; /* Seamless integration with existing white background */
}

/* Individual task field container - displays label and value inline */
.task-field {
    margin-bottom: 0.5rem; /* Spacing between field rows */
}

.task-field:last-child {
    margin-bottom: 0; /* Remove margin from last field to prevent extra space */
}

/* Field label styling - bold labels for "Subject:", "Due Date:", etc. */
.field-label {
    font-weight: 600; /* Semi-bold for emphasis */
    color: #3e3e3c; /* Salesforce standard text color for labels */
    margin-right: 0.25rem; /* Small gap between label and value */
}

/* Field value styling - actual data like "Call Ben", "8/5/2026", etc. */
.field-value {
    color: #181818; /* Salesforce standard text color for values */
}

.regen-btn .slds-button__icon {
    width: 2rem !important;    /* Default is 1rem */
    height: 2rem !important;
    font-size: 2rem !important;
    fill: #0176d3 !important;  /* Salesforce blue */
}

.button-row {
    display: flex;
    justify-content: space-between;
    align-items: center;
}

.relative-container {
    display: flex;                         
    position: relative;               
}

.thumb-btn {
    padding: 2px;    
    align-items: center; 
    justify-content: center; 
}

.thumb-btn .slds-button__icon { 
    width: 1rem;                
    height: 1rem;
}

/* Feedback Tooltip Popover Styles */
.custom-feedback-popover {
    position: absolute;
    top: calc(100% + 0.125rem);
    right: -0.5rem;
    z-index: 9001;
    width: 320px;
    max-width: 90vw;
    box-shadow: 0 2px 12px 0 rgba(0, 0, 0, 0.15);
    background-color: #ffffff;
    border: 1px solid #d8dde6;
    border-radius: 0.25rem;
}

/* Caret/Arrow pointing up to thumbs down icon */
.custom-feedback-popover::before {
    content: '';
    position: absolute;
    top: -0.5rem;
    right: 1rem;
    width: 0;
    height: 0;
    border-left: 0.5rem solid transparent;
    border-right: 0.5rem solid transparent;
    border-bottom: 0.5rem solid #d8dde6;
}

.custom-feedback-popover::after {
    content: '';
    position: absolute;
    top: -0.4375rem;
    right: 1rem;
    width: 0;
    height: 0;
    border-left: 0.5rem solid transparent;
    border-right: 0.5rem solid transparent;
    border-bottom: 0.5rem solid #ffffff;
}

.custom-feedback-popover .slds-popover__header {
    background-color: #ffffff;
    border-bottom: 1px solid #d8dde6;
    padding: 0.75rem 1rem;
}

.custom-feedback-popover .slds-popover__body {
    background-color: #ffffff;
    padding: 1rem;
}

.custom-feedback-popover .slds-popover__footer {
    background-color: #ffffff;
    border-top: 1px solid #d8dde6;
    padding: 0.75rem 1rem;
    display: flex;
    justify-content: flex-end;
    gap: 0.5rem;
}

.custom-feedback-popover .slds-form-element__legend {
    color: #3e3e3c;
    font-size: 0.875rem;
    font-weight: 400;
}

.custom-feedback-popover .slds-text-title_bold {
    color: #181818;
    font-size: 1rem;
    font-weight: 700;
}


.slds-checkbox {
    position: relative;
    display: block;
    margin-bottom: 0.5rem;
}

.slds-checkbox input[type="checkbox"] {
    position: absolute;
    opacity: 0;
    width: 1px;
    height: 1px;
    border: 0;
    clip: rect(0 0 0 0);
    margin: -1px;
    padding: 0;
    overflow: hidden;
}

.slds-checkbox__label {
    display: flex;
    align-items: flex-start;
    cursor: pointer;
    font-size: 0.875rem;
    line-height: 1.25;
}

.slds-checkbox__faux {
    width: 1rem;
    height: 1rem;
    display: inline-block;
    position: relative;
    flex-shrink: 0;
    border: 1px solid #c9c7c5;
    border-radius: 0.125rem;
    background: #fff;
    margin-right: 0.5rem;
    margin-top: 0.125rem;
}

.slds-checkbox input:checked + label .slds-checkbox__faux {
    background: #0176d3;
    border-color: #0176d3;
}

.slds-checkbox input:checked + label .slds-checkbox__faux::after {
    content: '';
    position: absolute;
    top: 0.125rem;
    left: 0.25rem;
    width: 0.25rem;
    height: 0.5rem;
    border: 2px solid #fff;
    border-top: 0;
    border-left: 0;
    transform: rotate(45deg);
}

.slds-form-element__legend {
    font-weight: 600;
    margin-bottom: 0.75rem;
}

.slds-modal__content {
    max-height: 60vh;
}

.custom-overflow{
    z-index: 1000;
}

.custom-heading-font{
    font-weight: bold;
}

.custom-helptext-padding{
    padding-top:14px
}

.custom-padding-left{
    padding-left: 100px;
}

.custom-textarea-font{
    color: #444444;
}
