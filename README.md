Meta can now predict what your brain is thinking. read that again.

TRIBE v2 scans how the brain responds to anything we see or hear. movies, music, speech. it creates a digital twin of neural activity and predicts our brain’s reaction without scanning us.

trained on 500+ hours of fMRI data from 700+ people. works on people it’s never seen before. no retraining needed. 2-3x more accurate than anything before it.

they also open-sourced everything. model weights, code, paper, demo. all of it. free.

the stated goal is neuroscience research and disease diagnosis. the unstated implication is that Meta now has a fucking foundation model that understands how our brains react to content/targetted ads

the company that sells our attention to advertisers just pulled out the psychology side of AI. we’re so cooked.
---
Today we're introducing TRIBE v2 (Trimodal Brain Encoder), a foundation model trained to predict how the human brain responds to almost any sight or sound.

Building on our Algonauts 2025 award-winning architecture, TRIBE v2 draws on 500+ hours of fMRI recordings from 700+ people to create a digital twin of neural activity and enable zero-shot predictions for new subjects, languages, and tasks.

Try the demo and learn more here:

---
https://aidemos.atmeta.com/tribev2/
---

private async _buildRefinerValueOptions() {
        const { [EventsService]: { refinersAsync } } = this.props.services;

        await refinersAsync.promise;

        const refiners = [...refinersAsync.data];
        refiners.sort(Refiner.OrderAscComparer);

        const refinerValueToDropdownOption = (value: RefinerValue) => {
            const { key, displayName: text } = value;
            return { key, text, data: value } as IDropdownOption;
        };

        const refinerValueOptionsByRefiner = new Map<Refiner, IDropdownOption[]>();
        for (const refiner of refiners) {
            const { required, allowMultiselect, blankValue } = refiner;
            const options: IDropdownOption[] = [];
            const panelRefinerValues = refiner.values
                .filter(Entity.NotDeletedFilter)
                .filter((value: RefinerValue) => (value as any).__external !== true);

            if (!required && !allowMultiselect) {
                options.push(refinerValueToDropdownOption(blankValue));
            }

            options.push(...panelRefinerValues.map(refinerValueToDropdownOption));

            refinerValueOptionsByRefiner.set(refiner, options);
        }

        this.setState({ refinerValueOptionsByRefiner, refiners });
    }
