<template>
    <lightning-card>
        <template if:true={isLoading}>
            <lightning-spinner alternative-text="loading..." size="medium"></lightning-spinner>
        </template>
        <lightning-tabset active-tab-value="generative">
            <lightning-tab label="Generative Actions" value="generative">
                <div class="main-content slds-p-around_medium">

                    <!-- PHASE 1: Before Generate -->
                    <template if:false={showReviewScreen}>
                    <p class="page-intro slds-m-bottom_medium">
                        Click 'Generate Actions' to let AI find recommended follow-up Actions.
                    </p>
                    <lightning-button 
                        label="Generate Actions"
                        variant="brand"
                        onclick={invokeAgentforce}
                        class="slds-m-bottom_x-large"
                    ></lightning-button>
                    </template>

                    <!-- PHASE 2: After Generate -->
                    <template if:true={showReviewScreen}>
                        <div class="review-info-msg slds-m-bottom_medium">
                            Here are AI recommended follow-up actions
                        </div>
                        <template for:each={visibleActionReviews} for:item="ar" for:index="idx">
                            <div key={ar.action.Id} class="review-container slds-box">
                                <lightning-button-icon
                                    icon-name="utility:close"
                                    alternative-text="Close"
                                    title="Close"
                                    variant="bare"
                                    size="medium"
                                    class="close-btn"
                                    data-action-id={ar.action.Id}
                                    onclick={handleCloseAction}>
                                </lightning-button-icon>
                                <!-- Card Title: Displays the main identifier for the follow-up action -->
                                <div class="card-title">{ar.cardTitle}</div>
                                
                                <!-- Task Details Section: Shows key task fields extracted from Agentforce data -->
                                <!-- Only displays when subject exists to avoid showing empty sections -->
                                <template if:true={ar.displaySubject}>
                                    <div class="task-details slds-m-top_small slds-m-bottom_small">
                                        <!-- Subject Field: Task title/description -->
                                        <div class="task-field">
                                            <span class="field-label">Subject:</span>
                                            <span class="field-value">{ar.displaySubject}</span>
                                        </div>
                                        <!-- Due Date Field: Task deadline (formatted for display) -->
                                        <div class="task-field">
                                            <span class="field-label">Due Date:</span>
                                            <span class="field-value">{ar.displayDueDate}</span>
                                        </div>
                                        <!-- Assigned To Field: Task owner (user ID or name) -->
                                        <div class="task-field">
                                            <span class="field-label">Assigned To:</span>
                                            <span class="field-value">{ar.displayAssignedTo}</span>
                                        </div>
                                        <!-- Related To Field: Associated record (account, contact, etc.) -->
                                        <div class="task-field">
                                            <span class="field-label">Related To:</span>
                                            <span class="field-value">{ar.displayRelatedTo}</span>
                                        </div>
                                    </div>
                                </template>
                                
                                <div class="button-row">
                                    <div class="button-area">
                                        <lightning-button
                                            label={ar.submitButtonLabel}
                                            variant="neutral"
                                            data-index={idx}
                                            onclick={handleSubmit}>
                                        </lightning-button>
                                    </div>
                                    <div class="relative-container">
                                        <!-- Thumbs Up Button: Shows success toast when clicked -->
                                        <lightning-button-icon
                                            icon-name="utility:like"
                                            alternative-text={thumbsUpLabel}
                                            title={thumbsUpLabel}
                                            variant="container"
                                            size="medium"
                                            data-action-id={ar.action.Id}
                                            onclick={handleThumbsUp}
                                            class="thumb-btn"
                                            >
                                        </lightning-button-icon>
                                        <!-- Thumbs Down Button: Opens detailed feedback modal when clicked -->
                                        <!-- Button turns blue when modal is open using dynamic thumbsDownClass -->

                                            <lightning-button-icon
                                                icon-name="utility:dislike"
                                                alternative-text={thumbsDownLabel}
                                                title={thumbsDownLabel}
                                                variant="container"
                                                data-action-id={ar.action.Id}
                                                onclick={handleThumbsDown}
                                                class="thumb-btn">
                                            </lightning-button-icon>
                                            <section class="slds-popover slds-popover_tooltip custom-feedback-popover custom-overflow slds-hide" 
                                                role="dialog" data-pop-id={ar.action.Id}>
                                                        <div class="relative-container">
                                                            <span class="slds-p-top_medium slds-p-left_medium slds-p-right_medium custom-heading-font">
                                                                <span class="slds-required">*</span>{whyNotHelpful} </span>                                                            
                                                                <lightning-helptext class="custom-helptext-padding slds-p-bottom_small" content={feedbackHelpText}></lightning-helptext>
                                                                <lightning-button-icon
                                                                    icon-name="utility:close"
                                                                    alternative-text={closeLabel}
                                                                    title={closeLabel}
                                                                    variant="bare"
                                                                    size="small"
                                                                    data-action-id={ar.action.Id}
                                                                    class="slds-float_right custom-padding-left slds-p-top_x-small"
                                                                    onclick={handleCancelFeedback}>
                                                                </lightning-button-icon>
                                                        </div>
                                                <div class="slds-popover__body">
                                                    <fieldset class="slds-form-element">
                                                        <div class="slds-form-element__control">
                                                            <template for:each={feedbackOptions} for:item="option">
                                                                <div key={option.value} class="relative-container">
                                                                    <lightning-input type="checkbox" variant="label-hidden" name={option.value} data-checkbox-id={option.value} value={option.value} onchange={handleCheckboxChange}></lightning-input>
                                                                    <label class="slds-form-element__label" for={option.value}>{option.label}</label>
                                                                </div>
                                                            </template>
                                                        </div>
                                                    </fieldset>
                                                    <label class="slds-form-element__label custom-heading-font slds-p-top_small" for="feedbackTextarea">{tellUsMore}</label>
                                                    <lightning-textarea name="feedbackTextarea" class="custom-textarea-font" variant="label-hidden" placeholder={feedbackPlaceholder} value={feedbackText} onchange={handleFeedbackTextChange}></lightning-textarea>
                                                </div>
                                                <footer class="slds-popover__footer">
                                                    <button class="slds-button slds-button_neutral" data-action-id={ar.action.Id} onclick={handleCancelFeedback}>{cancelLabel}</button>
                                                    <button class="slds-button slds-button_brand" disabled={isSubmitDisabled} onclick={handleSubmitFeedback}>{submitLabel}</button>
                                                </footer>
                                            </section>
                                    </div>
                                </div>
                            </div>
                        </template>
                        <lightning-button 
                            icon-name="utility:sparkles"
                            label="Regenerate"
                            variant="brand"
                            onclick={invokeAgentforce}
                            class="regen-btn"
                        ></lightning-button>
                    </template>
                    <template if:true={submitError}>
                        <div class="slds-box slds-theme_error slds-m-top_medium error-banner">
                            <span>{submitError}</span>
                        </div>
                    </template>
                </div>
            </lightning-tab>
        </lightning-tabset>
    </lightning-card>
</template>
