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
