import { LightningElement, api, wire, track } from 'lwc';
import askAgentInvocable from '@salesforce/apex/AF_AgentInvocationService.AF_InvokeAgent';
import getMeetingNoteActionReviewDetails from '@salesforce/apex/AF_MeetingNoteActionHandler.getMeetingNoteActionReviewDetails';
import getTask from '@salesforce/apex/AF_TaskDataController.getTask';
import runAction from '@salesforce/apex/AF_MeetingNoteActionRunner.runAction';
import { subscribe, unsubscribe } from 'lightning/empApi';
import { ShowToastEvent } from 'lightning/platformShowToastEvent';
import CreateOpportunity from 'c/afCreateOpportunityModal';
import markActionAsCancelled from '@salesforce/apex/AF_MeetingNoteActionHandler.markActionAsCancelled';
import getUserNameById from '@salesforce/apex/AF_TaskManageAction.getUserNameById';

// Custom Labels for Thumbs Up/Down Functionality
import THANKS_FOR_FEEDBACK from '@salesforce/label/c.af_thanks_for_feedback';
import THUMBS_UP_LABEL from '@salesforce/label/c.af_thumbs_up';
import THUMBS_DOWN_LABEL from '@salesforce/label/c.af_thumbs_down';
import WHY_NOT_HELPFUL from '@salesforce/label/c.af_why_not_helpful';
import FEEDBACK_HELP_TEXT from '@salesforce/label/c.af_feedback_help_text';
import TELL_US_MORE from '@salesforce/label/c.af_tell_us_more';
import FEEDBACK_PLACEHOLDER from '@salesforce/label/c.af_feedback_placeholder';
import CLOSE_LABEL from '@salesforce/label/c.af_close';
import CANCEL_LABEL from '@salesforce/label/c.af_cancel';
import SUBMIT_LABEL from '@salesforce/label/c.af_feedback_submit';
import FEEDBACK_INACCURATE from '@salesforce/label/c.af_feedback_inaccurate';
import FEEDBACK_INCOMPLETE from '@salesforce/label/c.af_feedback_incomplete';
import FEEDBACK_BIASED from '@salesforce/label/c.af_feedback_biased';
import FEEDBACK_INAPPROPRIATE from '@salesforce/label/c.af_feedback_inappropriate';
import FEEDBACK_OTHER from '@salesforce/label/c.af_feedback_other';

export default class AFAgentforceFollowUpActions extends LightningElement {
    @api recordId;
    @track task;
    @track error;
    description = '';
    @track hiddenActionIds = [];
    response;
    @api channelName = '/event/Task_Updated__e';
    subscription = null;
    @track isLoading = false;
    @track actionReviews = [];
    @track showReviewScreen = false;
    @track createdRecordLabel;
    @track createdRecordId;
    @track submitSuccess = false;
    @track submitError;  
    @track feedbackOptions = [ 
        { label: FEEDBACK_INACCURATE, value: 'inaccurate', checked: false },
        { label: FEEDBACK_INCOMPLETE, value: 'incomplete', checked: false },
        { label: FEEDBACK_BIASED, value: 'biased', checked: false },
        { label: FEEDBACK_INAPPROPRIATE, value: 'inappropriate', checked: false },
        { label: FEEDBACK_OTHER, value: 'other', checked: false }
    ];
@track feedbackText = ''; 
@track currentActionId = '';

// Simple textarea-style approach for checkboxes - only need button state
@track hasSelectedCheckbox = false;     // For submit button state

// Expose custom labels as properties for template access
get thanksForFeedback() { return THANKS_FOR_FEEDBACK; }
get thumbsUpLabel() { return THUMBS_UP_LABEL; }
get thumbsDownLabel() { return THUMBS_DOWN_LABEL; }
get whyNotHelpful() { return WHY_NOT_HELPFUL; }
get feedbackHelpText() { return FEEDBACK_HELP_TEXT; }
get tellUsMore() { return TELL_US_MORE; }
get feedbackPlaceholder() { return FEEDBACK_PLACEHOLDER; }
get closeLabel() { return CLOSE_LABEL; }
get cancelLabel() { return CANCEL_LABEL; }
get submitLabel() { return SUBMIT_LABEL; }

// Submit button state - disabled until checkbox selected
get isSubmitDisabled() {
    return !this.hasSelectedCheckbox;
}

    @wire(getTask, { recordId: '$recordId', description: '$description' })
    wiredTask(result) {
        this.wiredTaskData = result;
        if (result.data) {
            this.task = result.data;
            this.description = result.data.Description || '';
        } else if (result.error) {
            this.error = result.error;
        }
    }

    connectedCallback() {
        this.fetchAndDisplayActions();
        this.subscribeToPlatformEvent();
    }

    // Used both on page load and after generating actions
    fetchAndDisplayActions() {
        this.isLoading = true;
        getMeetingNoteActionReviewDetails({ taskId: this.recordId })
            .then(result => {
                this.processActionData(result);
                this.isLoading = false;
            })
            .catch(() => {
                this.actionReviews = [];
                this.showReviewScreen = false;
                this.isLoading = false;
            });
    }

    async processActionData(result) {
        if (result && result.length > 0) {
            // Using reference code pattern exactly as lead provided
            this.actionReviews = await Promise.all((result || []).map(async ar => ({
                ...ar,
                fields: await this.decorateFieldsForDisplay(ar.fields),
                // New functionality: Add object type detection for field display
                objectType: ar.action.AF_Object_Name__c?.toLowerCase() || '',
                isTaskObject: (ar.action.AF_Object_Name__c?.toLowerCase() === 'task'),
                isOpportunityObject: (ar.action.AF_Object_Name__c?.toLowerCase() === 'opportunity'),
                // New functionality: Add specific field display properties
                ...this.getFieldDisplayProperties(ar.fields, ar.action.AF_Object_Name__c)
            })));
            this.showReviewScreen = true;
        } else {
            this.actionReviews = [];
            this.showReviewScreen = false;
            if (this.agentInvoked === true) {
                this.showNoRecommendations = true;
            }
        }
    }

    // Using reference code method exactly as provided by lead
    async decorateFieldsForDisplay(fields) {
        return Promise.all(fields.map(async f => {
            const field = { ...f };
            
            // Reference code pattern: Optionally replace OwnerId with user name
            if (field.apiName === 'OwnerId' && field.value) {
                field.label = 'Assigned To';
                try {
                    const userName = await getUserNameById({ userId: field.value });
                    field.value = userName;
                } catch (e) { /* ignore */ }
            }
            
            // Reference code pattern: If it's a date type or field name/label has 'date', format it
            if (
                (field.type && field.type.toLowerCase() === 'date') ||
                (field.apiName && field.apiName.toLowerCase().includes('date')) ||
                (field.label && field.label.toLowerCase().includes('date'))
            ) {
                field.displayValue = this.formatDateForDisplay(field.value);
            }
            
            // New functionality: Enhanced for Opportunity Amount formatting
            if (field.apiName === 'Amount' && field.value) {
                field.displayValue = this.formatCurrency(field.value);
            }
            
            // New functionality: AF_Product_group__c handling (commented for this org)
            // if (field.apiName === 'AF_Product_group__c' && field.value) {
            //     field.displayValue = field.value;
            // }
            
            // Reference code pattern: Always set .uiValue to display in template
            field.uiValue = field.displayValue || field.value;
            
            return field;
        }));
    }

    // New functionality: Extract specific field values for Task/Opportunity display
    getFieldDisplayProperties(fields, objectType) {
        const objectTypeLower = objectType?.toLowerCase() || '';
        const fieldMap = {};
        
        // Build field lookup map
        fields.forEach(f => {
            fieldMap[f.apiName] = f;
        });
        
        const displayProps = {
            cardTitle: 'Follow-Up Action',
            submitButtonLabel: 'Submit'
        };
        
        if (objectTypeLower === 'task') {
            displayProps.submitButtonLabel = 'Create Task';
            displayProps.hasTaskFields = true;
            
            // Task-specific field values using reference code uiValue pattern
            displayProps.subjectValue = fieldMap['Subject']?.value || 'No subject';
            displayProps.dueDateValue = fieldMap['ActivityDate']?.displayValue || fieldMap['ActivityDate']?.value || 'No due date';
            displayProps.assignedToValue = fieldMap['OwnerId']?.value || 'Not assigned';
            displayProps.relatedToValue = fieldMap['WhatId']?.value || 'Not related';
            
            // Set card title from subject
            if (displayProps.subjectValue !== 'No subject') {
                displayProps.cardTitle = displayProps.subjectValue;
            }
            
        } else if (objectTypeLower === 'opportunity') {
            displayProps.submitButtonLabel = 'Create Opportunity';
            displayProps.hasOpportunityFields = true;
            
            // Opportunity-specific field values using reference code uiValue pattern
            displayProps.nameValue = fieldMap['Name']?.value || 'No name';
            displayProps.amountValue = fieldMap['Amount']?.displayValue || fieldMap['Amount']?.value || 'No amount';
            displayProps.clientNameValue = fieldMap['AF_Client_Name__c']?.value || 'No client';
            // displayProps.productGroupValue = fieldMap['AF_Product_group__c']?.value || 'No product group'; // For final org
            
            // Set card title from name
            if (displayProps.nameValue !== 'No name') {
                displayProps.cardTitle = displayProps.nameValue;
            }
        }
        
        return displayProps;
    }

    /**
     * Determines the appropriate submit button label based on object type
     * 
     * @param {String} objectApiName - API name of the Salesforce object
     * @returns {String} Button label text
     */
    getSubmitButtonLabel(objectApiName) {
        const apiName = objectApiName?.toLowerCase();
        
        switch (apiName) {
            case 'task':
                return 'Create Task';
            case 'opportunity':
                return 'Create Opportunity';
            default:
                return 'Submit';
        }
    }

    /**
     * Formats date values for consistent display in the UI
     * Enhanced version from reference code - converts YYYY-MM-DD to MM-DD-YYYY
     * 
     * @param {String} val - Date string from Salesforce
     * @returns {String} Formatted date string for display
     */
    formatDateForDisplay(val) {
        if (!val) return '';
        if (/^\d{4}-\d{2}-\d{2}$/.test(val)) {
            const [year, month, day] = val.split('-');
            return `${month}-${day}-${year}`;
        }
        return val;
    }

    /**
     * Formats currency values for Opportunity Amount display
     * 
     * @param {Number} amount - Currency amount
     * @returns {String} Formatted currency string
     */
    formatCurrency(amount) {
        if (!amount && amount !== 0) return '';
        try {
            return new Intl.NumberFormat('en-US', {
                style: 'currency',
                currency: 'USD'
            }).format(amount);
        } catch (error) {
            return amount.toString();
        }
    }

    /**
     * Gets readable name for WhatId (Related To field)
     * Placeholder method - would need actual implementation based on object types
     * 
     * @param {String} recordId - Record ID to resolve
     * @returns {String} Record name or fallback text
     */
    async getRelatedRecordName(recordId) {
        if (!recordId) return 'Not related';
        
        // TODO: Implement actual record name resolution
        // Would need to determine object type and query appropriate name field
        // For now, return the ID as fallback
        return recordId;
    }

    invokeAgentforce() {
        this.submitSuccess = false;
        this.submitError = undefined;
        this.createdRecordLabel = undefined;
        this.createdRecordId = undefined;
        this.isLoading = true;

        askAgentInvocable({ taskId: this.recordId, prompt: this.description })
            .then(res => {
                let parsedObj = res;
                if (typeof parsedObj === 'string') {
                    try { parsedObj = JSON.parse(parsedObj); }
                    catch (e) {
                        this.response = 'Agentforce response could not be parsed.';
                        this.isLoading = false;
                        return;
                    }
                }
                this.response = parsedObj && parsedObj.value ? parsedObj.value : 'Agent did not respond as expected.';

                // Only fetch action review details here, *not* in both load and generate
                this.fetchAndDisplayActions();
            })
            .catch(err => {
                this.isLoading = false;
                this.response = 'Agentforce call failed: ' + (err?.body?.message || err.message || err);
            });
    }

    async handleSubmit(event) {
        const idx = event.target.dataset.index;
        const actionReview = this.actionReviews[idx];
        if (!actionReview) return;

        // Use actionReview.otherFields for modal population (future)
        // For now, submit all fields
        const fields = {};
        (actionReview.fields || []).forEach(fld => {
            fields[fld.apiName] = fld.value !== '' && fld.value !== undefined ? fld.value : null;
        });

        const objectApiName = actionReview.action.AF_Object_Name__c;
        const actionId = actionReview.action.Id;
        console.log('event.target.label: ' + event.target.label);
        if (event.target.label === 'Create Opportunity') {
        this.openOpportunityModal(fields, actionId);  
        return;
        }
        this.submitError = undefined;
        this.isLoading = true;
  
        try {
            const result = await runAction({
                objectApiName: objectApiName,
                fieldValues: fields,
                taskId: this.recordId,
                actionId: actionId
            });

            if (result.errorMessage) {
                this.submitError = result.errorMessage;
                this.submitSuccess = false;
            } else {
                this.createdRecordLabel = result.objectLabel;
                this.createdRecordId = result.recordId;
                this.submitSuccess = true;
            }

            const toast = new ShowToastEvent({
                title: '',
                message: `Success! ${this.createdRecordLabel} has been created. {0}`,
                messageData: [
                    {
                        url: this.createdRecordUrl,
                        label: `View ${this.createdRecordLabel}`
                    }
                ],
                variant: 'success',
                mode: 'sticky'
            });

            this.dispatchEvent(toast);

        } catch (error) {
            this.submitError = error.body ? error.body.message : error.message;
            this.submitSuccess = false;
        }

        this.isLoading = false;
    }

    get createdRecordUrl() {
        return this.createdRecordId ? `/${this.createdRecordId}` : '#';
    }

    disconnectedCallback() {
        this.unsubscribeFromPlatformEvent();
    }

    subscribeToPlatformEvent() {
        subscribe(this.channelName, -1, (event) => {
            this.refreshMyData();
        })
            .then(response => {
                this.subscription = response;
            })
            .catch(error => {
                // Handle error if needed
            });
    }

    unsubscribeFromPlatformEvent() {
        if (this.subscription) {
            unsubscribe(this.subscription, () => {
                this.subscription = null;
            });
        }
    }

    refreshMyData() {
        // No change here, for wire refresh
        if (this.wiredTaskData && this.wiredTaskData.refresh) {
            this.wiredTaskData.refresh();
        }
    }

    handleCloseAction(event) {
        const actionId = event.target.dataset.actionId;
        if (!actionId) return;

        markActionAsCancelled({ actionId: actionId })
            .then(() => {
                this.hiddenActionIds = [...this.hiddenActionIds, actionId];
                this.dispatchEvent(
                    new ShowToastEvent({
                        title: 'Closed',
                        message: 'Action has been cancelled.',
                        variant: 'success'
                    })
                );
            })
            .catch((error) => {
                this.dispatchEvent(
                    new ShowToastEvent({
                        title: 'Error cancelling action',
                        message: error.body ? error.body.message : error.message,
                        variant: 'error'
                    })
                );
            });
    }

    get visibleActionReviews() {
        return this.actionReviews.filter(
            ar => !this.hiddenActionIds.includes(ar.action.Id)
        );
    }

    openOpportunityModal(fields, actionId) {
        console.log('openOpportunityModal');
    // Dynamic load if not already statically imported
        CreateOpportunity.open({
            size: 'medium',
            fields: fields,
            actionId: actionId
       
        });
    }

    // When user clicks thumbs up, show a quick success message and that's it
    // Uses 'dismissible' mode for close icon with auto-dismiss
    handleThumbsUp(event) {
        const actionId = event.target.dataset.actionId; 
        
        // Show toast with close icon
        this.dispatchEvent(
            new ShowToastEvent({
                title: THANKS_FOR_FEEDBACK,
                message: '',
                variant: 'success',
                mode: 'dismissible'     
            })
        );
        
        // Auto-dismiss after 3 seconds
        setTimeout(() => {
            // Dispatch a custom event to close the toast
            this.dispatchEvent(new CustomEvent('closetoast'));
        }, 3000);
    }

    // When user clicks thumbs down, we need more information, so open the modal
    // This also triggers the CSS class change to make button blue
    handleThumbsDown(event) {
        this.currentActionId = event.target.dataset.actionId;
        let targetActionId =  event.target.dataset.actionId;                   
        const target = this.template.querySelector(`[data-pop-id="${targetActionId}"]`);
        target.classList.remove('slds-hide');
        target.classList.add('slds-show');
        this.resetFeedbackForm();                        
    }

    // Utility method to reset the feedback form to initial clean state
    // Unchecks all checkboxes and clears the text area
    resetFeedbackForm() {
        this.template.querySelectorAll('lightning-input').forEach(element => {
            if(element.type === 'checkbox' || element.type === 'checkbox-button'){
                element.checked = false;
            }else{
                element.value = null;
            }      
        });
        this.feedbackText = '';
        this.hasSelectedCheckbox = false;
    }

    // Reusable method to read checkbox values from DOM - eliminates duplication
    getSelectedCheckboxValues() {
        const lightningInputs = this.template.querySelectorAll('lightning-input');
        const checkedValues = [];
        
        lightningInputs.forEach((input) => {
            if (input.type === 'checkbox' && input.checked) {
                checkedValues.push(input.value);
            }
        });
        
        return checkedValues;
    }

    // Simple textarea-style approach for checkboxes - only updates button state
    handleCheckboxChange(event) {
        // Use setTimeout to ensure DOM is updated
        setTimeout(() => {
            const checkedValues = this.getSelectedCheckboxValues();
            this.hasSelectedCheckbox = checkedValues.length > 0;
        }, 0);
    }

    // Captures user's additional feedback text as they type in the textarea
    handleFeedbackTextChange(event) {
        this.feedbackText = event.target.value; 
    }

    // Processes and submits the complete feedback when user clicks Submit
    // Collects both checkbox selections and text input
    handleSubmitFeedback() {
        // Use reusable method - eliminates duplication
        const checkedValues = this.getSelectedCheckboxValues();
        
        // Submit with captured values
        console.log('Feedback submitted:', {
            actionId: this.currentActionId,        
            selectedOptions: checkedValues,  // Checkbox values
            feedbackText: this.feedbackText  // Textarea value
        });

        // Store the current action ID before closing modal
        const actionIdToClose = this.currentActionId;

        // Show confirmation message to user - same style as thumbs up with close icon
        this.dispatchEvent(
            new ShowToastEvent({
                title: THANKS_FOR_FEEDBACK,
                message: '',
                variant: 'success',
                mode: 'dismissible'
            })
        );
        
        // Auto-dismiss after 3 seconds
        setTimeout(() => {
            // Dispatch a custom event to close the toast
            this.dispatchEvent(new CustomEvent('closetoast'));
        }, 3000);

        // Clean up and close the modal with proper action ID
        this.closeFeedbackModal(actionIdToClose);
    }

    // When user clicks Cancel button, just close modal without saving anything
    handleCancelFeedback(event) {
        const targetCloseId = event.target.dataset.actionId;
        this.closeFeedbackModal(targetCloseId); 
    }

    // Centralized method to close modal and reset all related state
    // This also triggers the thumbs down button to return to gray color
    closeFeedbackModal(targetCloseId) {
        const target = this.template.querySelector(`[data-pop-id="${targetCloseId}"]`);
        target.classList.add('slds-hide');
        target.classList.remove('slds-show');
        this.currentActionId = '';      
        this.resetFeedbackForm(); 
    }

}
