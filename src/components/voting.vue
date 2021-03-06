<template>
	<div id="voting">
		<h3>{{ title[status] }}</h3>
		<div class="vote-wrapper">
			<div v-for="vote in votes" :key="vote.id" class="vote-item">
				<p class="metadata">
					<span>#{{ vote.id }}:</span><span>{{ vote.metadata }}</span>
				</p>
				<div v-if="vote.state == 0">
					<span
						class="yes green"
						@click="toVote({ id: vote.id, supports: true })"
						>YES</span
					>
					<span class="no red" @click="toVote({ id: vote.id, supports: false })"
						>NO</span
					>
				</div>
				<p class="green" v-if="vote.state == 1">
					You voted YES
				</p>
				<p class="red" v-if="vote.state == 2">You voted No</p>
			</div>
		</div>
	</div>
</template>
<script>
import connect from "@aragon/connect"
import connectVoting from "@aragon/connect-voting"

export default {
	mounted() {
		this.init()
	},
	data() {
		return {
			title: {
				"0": "Loading votes...",
				"-1": "Load failed",
				"2": "Votes:",
			},
			status: "0", // 0 loading; -1 falied -2 success
			votes: [],
		}
	},
	methods: {
		async init() {
			try {
				const org = await connect("0ops.aragonid.eth", "thegraph", {
					network: flamincome.__net_id__,
				})

				// Connect the Voting app using the corresponding connector:
				const voting = await connectVoting(org.app("voting"))
				// Fetch votes of the Voting app
				let votes = await voting.votes()
				this.votes = votes.filter((vote) => {
					if (vote.executed === false) {
						vote.id = vote.id.substr(
							vote.id.indexOf("voteId:0x") + "voteId:0x".length
						)
						vote.id = new web3.utils.BN(vote.id, 16).toString()
						this.getVoterState(vote)
						return true
					} else return false
				})
				this.status = "1"
			} catch (error) {
				console.error(error)
				this.status = "-1"
			}
		},
		getVoterState(vote) {
			const votingContract = flamincome.__get_voting__()
			votingContract.methods
				.getVoterState(new web3.utils.BN(vote.id), flamincome.__account__)
				.call()
				.then((res) => {
					this.$set(vote, "state", res)
				})
				.catch((err) => {
					console.error(err)
				})
		},
		toVote({ id, supports, executes = true }) {
			flamincome.__display__("Ready to vote...")
			flamincome.__before__(() => {
				flamincome.__check_connection__()
				try {
					console.log(new web3.utils.BN(id).toNumber())
					console.log(supports)
					console.log(executes)
					const votingContract = flamincome.__get_voting__()
					flamincome.__transaction__(
						votingContract.methods
							.vote(new web3.utils.BN(id).toNumber(), supports, executes)
							.send({ from: flamincome.__account__ })
					)
				} catch (error) {
					flamincome.__display__(error.message)
					flamincome.__done__()
				}
			})
		},
	},
}
</script>
<style scoped>
.vote-wrapper {
	display: grid;
	grid-template-columns: repeat(auto-fill, 200px);
	grid-template-rows: repeat(auto-fill, 140px);
	grid-column-gap: 10px;
}
.vote-item {
	display: flex;
	margin-bottom: 5px;
	padding: 5px;
	flex-direction: column;
	border: dotted;
}
.metadata {
	height: 100px;
	overflow: hidden;
}
.green {
	color: rgb(14, 174, 106);
}
.red {
	color: salmon;
}
.yes {
	cursor: pointer;
	text-decoration: underline;
	margin-right: 10px;
}
.no {
	cursor: pointer;
	text-decoration: underline;
}
</style>
