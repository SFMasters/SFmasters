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



@track feedbackOptions = [ 
        { label: FEEDBACK_INACCURATE, value: 'inaccurate', checked: false },
        { label: FEEDBACK_INCOMPLETE, value: 'incomplete', checked: false },
        { label: FEEDBACK_BIASED, value: 'biased', checked: false },
        { label: FEEDBACK_INAPPROPRIATE, value: 'inappropriate', checked: false },
        { label: FEEDBACK_OTHER, value: 'other', checked: false }
    ];
    @track feedbackText = ''; 
    @track currentActionId = '';




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
        // this.feedbackOptions = this.feedbackOptions.map(option => ({
        //     ...option,
        //     checked: false 
        // }));
        this.template.querySelectorAll('lightning-input').forEach(element => {
        if(element.type === 'checkbox' || element.type === 'checkbox-button'){
            element.checked = false;
        }else{
            element.value = null;
        }      
        });
        this.feedbackText = '';
    }

    // Handles when user checks/unchecks any of the 5 predefined feedback options
    // Updates the feedbackOptions array to track which boxes are checked
    handleFeedbackOptionChange(event) {
        const value = event.target.value; 
        this.feedbackOptions = this.feedbackOptions.map(option => ({
            ...option,
            checked: option.value === value ? event.target.checked : option.checked
        }));
    }

    // Captures user's additional feedback text as they type in the textarea
    handleFeedbackTextChange(event) {
        this.feedbackText = event.target.value; 
    }

    // Processes and submits the complete feedback when user clicks Submit
    // Collects both checkbox selections and text input
    handleSubmitFeedback() {
        // Extract all checked options into an array of labels
        const selectedOptions = this.feedbackOptions
            .filter(option => option.checked)      
            .map(option => option.label);         

        // For now, we just log to console for demonstration
        console.log('Feedback submitted:', {
            actionId: this.currentActionId,        
            selectedOptions: selectedOptions,
            feedbackText: this.feedbackText 
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
