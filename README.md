// US-58: PersonLifeEvent filtering logic - Preserve original Lead/null behavior, only update Contact logic
                if (request.objectApiName == 'PersonLifeEvent') {
                    System.debug('PersonLifeEvent objectApiName: ' + request.objectApiName);
                    
                    // Original behavior: Always skip for Lead WhoId or null WhoId (regardless of WhatId)
                    if ((whoId != null && whoId.startsWith('00Q')) || whoId == null) {
                        skip = true;
                        skipMessage = 'Skipped: WhoId is a Lead. Action not created.';
                        System.debug('PersonLifeEvent skipMessage: ' + skipMessage);
                    } else if (whoId != null && whoId.startsWith('003')) {
                        // Updated Contact logic: Check WhatId for Contact-only scenario
                        Task sourceTask = sourceTaskMap.get(request.taskId);
                        
                        if (sourceTask != null && sourceTask.WhatId == null) {
                            // Contact-only scenario - WhoId is Contact (003xxx) and WhatId is null
                            skip = true;
                            skipMessage = 'Skipped: PersonLifeEvent creation not allowed for Contact-only meeting notes';
                            System.debug('*** SKIPPING PERSONLIFEEVENT *** - Contact-only meeting note. Task ID: ' + sourceTask.Id);
                        } else {
                            // Contact with WhatId - proceed with duplicate check
                            // Extract EventType and EventDate from JSON for duplicate check
                            List<Object> fieldList = (List<Object>) JSON.deserializeUntyped(request.fieldApiNameToValueJSON);
                            for (Object field : fieldList) {
                                Map<String, Object> fieldMap = (Map<String, Object>)field;
                                if (fieldMap.containsKey('EventType')) {
                                    eventType = String.valueOf(fieldMap.get('EventType'));
                                }
                                if (fieldMap.containsKey('EventDate')) {
                                    eventDateStr = String.valueOf(fieldMap.get('EventDate'));
                                }
                            }

                            // Duplicate check for existing PersonLifeEvent
                            if (eventType != '' && eventDateStr != '') {
                                Date eventDate;
                                try { 
                                    eventDate = Date.valueOf(eventDateStr); 
                                } catch(Exception e) { 
                                    eventDate = null; 
                                }
                                if (eventDate != null) {
                                    List<PersonLifeEvent> foundEvents = [
                                        SELECT Id FROM PersonLifeEvent
                                        WHERE PrimaryPersonId = :whoId
                                          AND EventType = :eventType
                                          AND EventDate = :eventDate
                                        LIMIT 1
                                    ];
                                    if (!foundEvents.isEmpty()) {
                                        skip = true;
                                        skipMessage = 'Skipped: PersonLifeEvent already exists for this person, type, and date.';
                                        System.debug('*** SKIPPING PERSONLIFEEVENT *** - Duplicate found.');
                                    }
                                }
                            }
                        }
                    }
                }
				
				
				
				
				
				
				
				
				
				
				
				
				if (!taskIds.isEmpty()) {
                // Calculate dynamic limit based on remaining query rows to avoid governor limits
                Integer remainingQueryRows = Limits.getLimitQueryRows() - Limits.getQueryRows();
                Integer queryLimit = Math.min(remainingQueryRows, taskIds.size());
                
                if (queryLimit > 0) {
                    for (Task task : [SELECT Id, WhoId, WhatId
                                      FROM Task
                                      WHERE Id IN :taskIds
                                      LIMIT :queryLimit]) {
                        sourceTaskMap.put(task.Id, task);
                        System.debug('Found Task - ID: ' + task.Id + ', WhoId: ' + task.WhoId + ', WhatId: ' + task.WhatId);
                    }
                } else {
                    System.debug('WARNING: No query rows remaining. Cannot query tasks for filtering logic.');
                }
            }
            System.debug('Total Tasks queried: ' + sourceTaskMap.size());
