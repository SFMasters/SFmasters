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
