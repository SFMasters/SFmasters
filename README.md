    processActionData(result) {
        if (result && result.length > 0) {
            this.actionReviews = this.formatActionReviews(result);
            this.showReviewScreen = true;
        } else {
            this.actionReviews = [];
            this.showReviewScreen = false;
            if (this.agentInvoked === true) {
                this.showNoRecommendations = true;
            }
        }
    }

    formatActionReviews(actionResults) {
        return (actionResults || []).map(ar => {
            let subject = '';
            let opportunity = '';

            if (ar.fields && Array.isArray(ar.fields)) {
                ar.fields.forEach(f => {
                    if ((f.apiName === 'Subject' || f.label === 'Subject') && f.value) {
                        subject = f.value;
                    }
                    if ((f.apiName === 'Name' || f.label === 'Opportunity Name') && f.value) {
                        opportunity = f.value;
                    }
                });
            }

            let cardTitle = '';
            if (opportunity) { cardTitle = opportunity; }
            else if (subject) { cardTitle = subject; }
            else { cardTitle = reviewTitle; }

            let submitButtonLabel = 'Submit';
            const apiName = (ar.action && ar.action.AF_Object_Name__c)
                ? ar.action.AF_Object_Name__c.toLowerCase()
                : '';

            if (apiName === 'task') submitButtonLabel = reviewTask;
            else if (apiName === 'opportunity') submitButtonLabel = reviewOpportunity;

            const otherFields = (ar.fields || []).filter(f =>
                !((f.apiName === 'Subject' || f.label === 'Subject') ||
                (f.apiName === 'OpportunityName' || f.label === 'Opportunity Name'))
            );

            return { ...ar, cardTitle, otherFields, submitButtonLabel };
        });
    }

    async invokeAgentforce() {
        this.submitSuccess = false;
        this.submitError = undefined;
        this.createdRecordLabel = undefined;
