[`./consensus/src/consensus.rs`]

/// Spins the consensus state machine. The consensus runs for the whole
/// round until either a new round is produced or the node needs to re-sync.
///
/// The Agreement loop (acting roundwise) runs concurrently with the
/// generation-selection-reduction loop (acting step-wise).

fn *spin* :
    provisioners.update_eligibility_flag(ru.round);

    agreement_process.spawn(
        ru.clone(),
        provisioners.clone(),
        self.db.clone(),
    );

    spawn_main_loop(
            ru,
            provisioners,
            self.agreement_process.inbound_queue.clone(),
        );

fn *spawn_main_loop* :
    phases = [
                Phase::Selection(),
                Phase::Reduction1(),
                Phase::Reduction2(),
            ];
    
    step: u8 = 0;

    loop {
        for phase in phases {
            step += 1;

            phase.initialize(&msg, ru.round, step);
    }