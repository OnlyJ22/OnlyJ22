the GitHub of the dev who rewrote the code in rust/python https://github.com/ultraworkers/claw-code

---
nice overview site that breaks down the code https://ccunpacked.dev/#agent-loop

---
TRIBE v2 scans how the brain responds to anything we see or hear. movies, music, speech.

Today we're introducing TRIBE v2 (Trimodal Brain Encoder), a foundation model trained to predict how the human brain responds to almost any sight or sound.

Trained on 500+ hours of fMRI data from 700+ people. works on people it’s never seen before. no retraining needed. 2-3x more accurate than anything before it.

They also open-sourced everything. model weights, code, paper, demo. all of it. free.

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

    ----

    https://github.com/Yng808/sp-dev-fx-webparts/commit/48e9b69de
