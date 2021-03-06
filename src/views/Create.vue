<template>
  <Container :slim="true">
    <div class="px-4 px-md-0 mb-3">
      <router-link
        :to="{ name: domain ? 'home' : 'proposals' }"
        class="text-gray"
      >
        <Icon name="back" size="22" class="v-align-middle" />
        {{ space.name }}
      </router-link>
    </div>
    <div>
      <div class="col-12 col-lg-8 float-left pr-0 pr-lg-5">
        <div class="px-4 px-md-0">
          <div class="d-flex flex-column mb-6">
            <input
              v-autofocus
              v-model="form.name"
              maxlength="128"
              class="h1 mb-2 input"
              placeholder="Question"
            />
            <textarea-autosize
              v-model="form.body"
              maxlength="10240"
              class="input pt-1 mb-6"
              placeholder="What is your proposal?"
            />
            <div v-if="form.body">
              <h4 class="mb-4">Preview</h4>
              <UiMarkdown :body="form.body" />
            </div>
          </div>
        </div>
        <Block title="Choices">
          <div v-if="choices.length > 0" class="overflow-hidden mb-2">
            <draggable v-model="choices">
              <transition-group name="list">
                <div
                  v-for="(choice, i) in choices"
                  :key="choice.key"
                  class="d-flex mb-2"
                >
                  <UiButton class="d-flex width-full">
                    <span class="mr-4">{{ i + 1 }}</span>
                    <input
                      v-model="choices[i].text"
                      class="input height-full flex-auto text-center"
                    />
                    <span @click="removeChoice(i)" class="ml-4">
                      <Icon name="close" size="12" />
                    </span>
                  </UiButton>
                </div>
              </transition-group>
            </draggable>
          </div>
          <UiButton @click="addChoice(1)" class="d-block width-full">
            Add choice
          </UiButton>
        </Block>
      </div>
      <div class="col-12 col-lg-4 float-left">
        <Block
          title="Actions"
          :icon="space.network === '4' ? 'stars' : undefined"
          @submit="modalPluginsOpen = true"
        >
          <div class="mb-2">
            <UiButton
              @click="[(modalOpen = true), (selectedDate = 'start')]"
              class="width-full mb-2"
            >
              <span v-if="!form.start">Select start date</span>
              <span v-else v-text="$d(form.start * 1e3, 'short')" />
            </UiButton>
            <UiButton
              @click="[(modalOpen = true), (selectedDate = 'end')]"
              class="width-full mb-2"
            >
              <span v-if="!form.end">Select end date</span>
              <span v-else v-text="$d(form.end * 1e3, 'short')" />
            </UiButton>
            <div class="advanced-wrapper">
              <svg
                width="13"
                height="8"
                viewBox="0 0 13 8"
                fill="none"
                :class="{ arrow: true, enabled: showAdvaced }"
              >
                <path
                  d="M12 7L6.5 1L1 7"
                  stroke-width="2"
                  stroke-linecap="round"
                  stroke-linejoin="round"
                />
              </svg>
              <div class="advanced" @click="showAdvaced = !showAdvaced">
                Advanced
              </div>
            </div>
            <!-- <UiButton v-show="showAdvaced" class="width-full mb-2">
              <input
                v-model="form.snapshot"
                type="number"
                disabled
                class="input width-full text-center"
                placeholder="Snapshot block number"
              />
            </UiButton> -->
            <UiButton v-show="showAdvaced" class="width-full mb-2">
              <input
                v-model="form.quorum"
                min="0"
                max="100"
                type="number"
                class="input width-full text-center"
                placeholder="Snapshot quorum"
              />
            </UiButton>
          </div>
          <UiButton
            @click="handleSubmit"
            :disabled="!isValid"
            :loading="loading"
            class="d-block width-full button--submit"
          >
            Publish
          </UiButton>
        </Block>
      </div>
    </div>
    <ModalSelectDate
      :value="form[selectedDate]"
      :selectedDate="selectedDate"
      :open="modalOpen"
      @close="modalOpen = false"
      @input="setDate"
    />
    <ModalPlugins
      :proposal="{ ...form, choices }"
      :value="form.metadata.plugins"
      v-model="form.metadata.plugins"
      :open="modalPluginsOpen"
      @close="modalPluginsOpen = false"
    />
  </Container>
</template>

<script>
import { mapActions } from 'vuex';
import draggable from 'vuedraggable';
import { getBlockNumber, getTotalSupply } from '@/helpers/web3';

export default {
  components: {
    draggable
  },
  data() {
    return {
      key: this.$route.params.key,
      loading: false,
      showAdvaced: false,
      choices: [],
      blockNumber: -1,
      form: {
        name: '',
        body: '',
        choices: [],
        start: '',
        end: '',
        snapshot: 0,
        totalSupply: '0',
        quorum: 20,
        metadata: {}
      },
      modalOpen: false,
      modalPluginsOpen: false,
      selectedDate: '',
      counter: 0
    };
  },
  computed: {
    space() {
      return this.app.spaces[this.key];
    },
    isValid() {
      // const ts = (Date.now() / 1e3).toFixed();
      return (
        !this.loading &&
        this.web3.account.base16 &&
        this.form.name &&
        this.form.body &&
        this.form.start &&
        Number(this.form.quorum) <= 100 &&
        // this.form.start >= ts &&
        this.form.end &&
        this.form.end > this.form.start &&
        // this.form.snapshot &&
        // this.form.snapshot > this.blockNumber / 2 &&
        this.choices.length >= 2 &&
        !this.choices.some(a => a.text === '')
      );
    }
  },
  async mounted() {
    this.addChoice(2);

    this.blockNumber = await getBlockNumber(this.$auth.web3);
    this.totalSupply = await getTotalSupply(this.$auth.web3, this.space.token);
    this.form.snapshot = this.blockNumber;
  },
  methods: {
    ...mapActions(['send']),
    addChoice(num) {
      for (let i = 1; i <= num; i++) {
        this.counter++;
        this.choices.push({ key: this.counter, text: '' });
      }
    },
    removeChoice(i) {
      this.choices.splice(i, 1);
    },
    setDate(ts) {
      if (this.selectedDate) {
        this.form[this.selectedDate] = ts;
      }
    },
    async handleSubmit() {
      this.loading = true;
      this.form.choices = this.choices.map(choice => choice.text);
      try {
        const { ipfsHash } = await this.send({
          token: this.space.token,
          type: 'proposal',
          payload: this.form
        });
        this.$router.push({
          name: 'proposal',
          params: {
            key: this.key,
            id: ipfsHash
          }
        });
      } catch (e) {
        console.error(e);
        this.loading = false;
      }
    }
  }
};
</script>

<style>
.list-leave-active,
.list-enter-active {
  transition: all 0.3s;
}
.list-move {
  transition: transform 0.3s;
}
.list-enter,
.list-leave-to {
  opacity: 0;
}
.advanced {
  user-select: none;
  cursor: pointer;
  text-align: right;
  color: var(--heading-color);
}
.advanced-wrapper {
  display: flex;
  align-items: center;
  justify-content: flex-end;
}
.arrow {
  margin-bottom: 7px;
  margin-right: 2px;
  transform: rotate(180deg);
}
.arrow > path {
  stroke: var(--heading-color);
}
.enabled {
  transform: rotate(0);
}
</style>
