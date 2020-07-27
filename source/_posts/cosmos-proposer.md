---

title: cosmos-proposer
categories:
  - cosmos
date: 2019-12-19 15:44:55
tags:
---

## Tendermint Proposer

之前有提到`cosmos-staking`是决定更新验证者。

这里讲到的是这些验证者们如何选择出proposer。



```
func (vals *ValidatorSet) incrementProposerPriority() *Validator {
	for _, val := range vals.Validators {
		//计算每个验证者的ProposerPriority = ProposerPriority + VotingPower
		newPrio := safeAddClip(val.ProposerPriority, val.VotingPower)
		val.ProposerPriority = newPrio
	}
	//取出ProposerPriority最大的验证者来作为proposer
	// Decrement the validator with most ProposerPriority.
	mostest := vals.getValWithMostPriority()
	//选择出proposer后，其ProposerPriority会进行再更新为ProposerPriority - TotalVotingPower
	// Mind the underflow.
	mostest.ProposerPriority = safeSubClip(mostest.ProposerPriority, vals.TotalVotingPower())

	return mostest
}
```

大概说来，就是Priority = Priority + VotingPower，max(Priority)的验证者作为proposer，且更新其Priority = Priority - TotalVotingPower。

测试

```
func TestProposerSelection1(t *testing.T) {
	vset := NewValidatorSet([]*Validator{
		newValidator([]byte("foo"), 1000),
		newValidator([]byte("bar"), 300),
		newValidator([]byte("baz"), 330),
	})
	for _, v :=range vset.Validators  {
		fmt.Println(string(v.Address), v.ProposerPriority)
	}
	var proposers []string
	for i := 0; i < 20; i++ {
		val := vset.GetProposer()
		proposers = append(proposers, string(val.Address))
		fmt.Print("[", string(val.Address), "]",)
		for _, v :=range vset.Validators  {
			fmt.Print(string(v.Address),":", v.ProposerPriority, " / ")
		}
		fmt.Println()
		vset.IncrementProposerPriority(1)
	}
```

打印结果如下

```
[foo]bar:300 / baz:330 / foo:-630 / 
[baz]bar:600 / baz:-970 / foo:370 / 
[foo]bar:900 / baz:-640 / foo:-260 / 
[bar]bar:-430 / baz:-310 / foo:740 / 
[foo]bar:-130 / baz:20 / foo:110 / 
[foo]bar:170 / baz:350 / foo:-520 / 
[baz]bar:470 / baz:-950 / foo:480 / 
[foo]bar:770 / baz:-620 / foo:-150 / 
[bar]bar:-560 / baz:-290 / foo:850 / 
[foo]bar:-260 / baz:40 / foo:220 / 
[foo]bar:40 / baz:370 / foo:-410 / 
[baz]bar:340 / baz:-930 / foo:590 / 
[foo]bar:640 / baz:-600 / foo:-40 / 
[foo]bar:940 / baz:-270 / foo:-670 / 
[bar]bar:-390 / baz:60 / foo:330 / 
[foo]bar:-90 / baz:390 / foo:-300 / 
[baz]bar:210 / baz:-910 / foo:700 / 
[foo]bar:510 / baz:-580 / foo:70 / 
[foo]bar:810 / baz:-250 / foo:-560 / 
[bar]bar:-520 / baz:80 / foo:440 /
```



不过，上面的计算并不完整，还有其他转换。

```
func (vals *ValidatorSet) IncrementProposerPriority(times int) {
	if vals.IsNilOrEmpty() {
		panic("empty validator set")
	}
	if times <= 0 {
		panic("Cannot call IncrementProposerPriority with non-positive times")
	}

	// Cap the difference between priorities to be proportional to 2*totalPower by
	// re-normalizing priorities, i.e., rescale all priorities by multiplying with:
	//  2*totalVotingPower/(maxPriority - minPriority)
	diffMax := PriorityWindowSizeFactor * vals.TotalVotingPower()
	vals.RescalePriorities(diffMax)
	vals.shiftByAvgProposerPriority()

	var proposer *Validator
	// Call IncrementProposerPriority(1) times times.
	for i := 0; i < times; i++ {
		proposer = vals.incrementProposerPriority()
	}

	vals.Proposer = proposer
}


func (vals *ValidatorSet) RescalePriorities(diffMax int64) {
	if vals.IsNilOrEmpty() {
		panic("empty validator set")
	}
	// NOTE: This check is merely a sanity check which could be
	// removed if all tests would init. voting power appropriately;
	// i.e. diffMax should always be > 0
	if diffMax <= 0 {
		return
	}

	// Calculating ceil(diff/diffMax):
	// Re-normalization is performed by dividing by an integer for simplicity.
	// NOTE: This may make debugging priority issues easier as well.
	diff := computeMaxMinPriorityDiff(vals)
	ratio := (diff + diffMax - 1) / diffMax
	if diff > diffMax {
		for _, val := range vals.Validators {
			val.ProposerPriority /= ratio
		}
	}
}



func (vals *ValidatorSet) shiftByAvgProposerPriority() {
	if vals.IsNilOrEmpty() {
		panic("empty validator set")
	}
	avgProposerPriority := vals.computeAvgProposerPriority()
	for _, val := range vals.Validators {
		val.ProposerPriority = safeSubClip(val.ProposerPriority, avgProposerPriority)
	}
}

```

