<template>
  <div class="grid-crash">
    <aside-bar-component></aside-bar-component>
    <chat-component></chat-component>
    <header-component></header-component>
    <div class="crash-content">
      <section class="crash-game">
        <div class="crash-game__content">
          <div class="crash-game__start-window">
            <div class="classMenegerCrash">
              <h2 class="crash__game-name">Краш</h2>
              <div class="classNoMenegerCrash">
                <a href="https://docs.google.com/document/d/13jJx5IC0AC8pOpbSG5CU1i9tIQof0Vb0IUFzcR0aKcE" target="_blank">
                  <img src="@/assets/icons-games/double-game/info.svg" alt="" />
                </a>
              </div>
            </div>
            <p>Жди момента и забирай выигрыш</p>
            <div class="crash-game-window__inputs">
              <div class="crash-deposit">
                <h3 class="crash-label">Сумма депозита</h3>
                <div class="crash-input-deposit">
                  <img class="diamond-icon" src="@/assets/icons-games/saper-game/icon-diamond-ore-saper.png" />
                  <input class="input-style-crash input-style-crash__text diamond-input-crash" type="number" v-model="amountDeposit" />
                  <div class="diamonds__btns btn-style__diamonds">
                    <ul class="diamonds-btns__display">
                      <li v-for="(item, index) in SaperNumbers" :key="index">
                        <button
                            @click="clickedBtnChoice(index, item.diamonds)"
                            :class="{ 'btn-click': clickedBtn === index, [index]: clickedBtn === index }"
                            :id="item.diamonds === 'max' ? 'max-button' : null"
                            v-if="item.diamonds !== undefined"
                        >
                          {{ item.diamonds }}
                        </button>
                      </li>
                    </ul>
                  </div>
                </div>
              </div>
            </div>
            <div class="error-block" v-if="ErrorClick">
              <h2>{{ textError }}</h2>
            </div>
            <div class="crash-game-window__btn-start">
              <button
                  v-if="startGame && crashObject.Status !== 'WaitingForPlayers'"
                  @click="clickClaimDep"
                  class="claim-btn-crash__prize"
              >
                Забрать {{ getUserPrize.toFixed(2) }} АР
              </button>
              <button v-if="startGame && crashObject.Status === 'WaitingForPlayers'" class="waiting-btn-crash">
                Ожидание игроков...
              </button>
              <button v-if="!startGame" :disabled="offBtn" @click="clickPlayBtn" :class="{ 'animate-start-btn': ErrorClick }">
                Начать игру
              </button>
            </div>
            <div class="crash-window__line">
              <div class="line-crash"></div>
            </div>
          </div>
          <div class="crash-game__graph">
            <crash-graph-component :crashdata="crashObject"></crash-graph-component>
          </div>
        </div>
      </section>
      <section class="crash-game__players" v-if="crashObject && crashObject.Players">
        <div class="crash-game-players__content">
          <ul class="user-list" v-if="crashObject.Players.length">
            <li class="user-crash" v-for="(player, index) in crashObject.Players.sort((a, b) => b.Bid - a.Bid)" :key="index">
              <div
                  class="user-crash-content"
                  :class="{ 'user-crash-content__lose': crashObject.Status === 'GameEnd' && player.WinningX <= 0, 'user-crash-content__win': player.WinningX >= 1 }"
              >
                <div class="user-name-crash">
                  <img class="user-crash__icon" :src="`https://mc-heads.net/avatar/${player.UserName}/55`" />
                  <h2>{{ player.UserName }}</h2>
                </div>
                <div class="user-bid-crash">
                  <h2 v-if="player.WinningMoney >= 1" class="wingame__win-bid-crash">
                    {{ player.WinningMoney.toFixed(2) }}
                  </h2>
                  <h2 v-else>{{ player.Bid }}</h2>
                  <img src="@/assets/icons-games/saper-game/icon-diamond-ore-saper.png" />
                </div>
                <div class="user-game-status-crash">
                  <h2 class="ingame-crash" v-if="crashObject.Status === 'WaitingForPlayers' && player.WinningX <= 1 || crashObject.Status === 'InGame' && player.WinningX <= 0">
                    В игре
                  </h2>
                  <h2 class="lostgame-crash" v-if="crashObject.Status === 'GameEnd' && player.WinningX <= 0">
                    Проиграл
                  </h2>
                  <h2 class="wingame-crash" v-if="player.WinningX >= 1">{{ player.WinningX.toFixed(2) }}x</h2>
                </div>
              </div>
            </li>
          </ul>
        </div>
      </section>
    </div>
  </div>
</template>

<script>
import HeaderComponent from "@/components/HeaderComponent.vue";
import AsideBarComponent from "@/components/AsidebarComponent.vue";
import ChatComponent from "@/components/ChatComponent.vue";
import CrashGraphComponent from "@/components/games-components/CrashGraphComponent.vue";
import { useVuelidate } from "@vuelidate/core";
import { maxValue, minValue, required, numeric, integer } from "@vuelidate/validators";
import "@/assets/css/PagesStyles/games-pages/crash.css";
import SaperNumbers from "@/mocks/SaperNumbers";
import { GetCurrentMoney } from "@/assets/js/rest/RestMethods";
import { GetCookie } from "@/assets/js/storage/CookieStorage";
import { eventBus } from "@/main";
import { ExitAndTakeMoneyFromCrashGame, JoinCrashGame } from "@/assets/js/games/crash/CrashAPI";

export default {
  components: { HeaderComponent, AsideBarComponent, ChatComponent, CrashGraphComponent },
  data() {
    return {
      SaperNumbers,
      clickedBtn: null,
      ErrorClick: false,
      ErrorJoin: false,
      balance: 0,
      amountDeposit: 0,
      crashObject: "",
      textError: "",
      startGame: false,
      offBtn: false,
      isClaiming: false,
    };
  },
  setup() {
    return { v$: useVuelidate() };
  },
  computed: {
    getUserPrize() {
      const user = this.crashObject.Players.find(
          (player) => player.UserName === GetCookie("SpUserName")
      );
      return user ? user.Bid * this.crashObject.CurrentX : 0;
    },
  },
  validations() {
    return {
      amountDeposit: { required, numeric, minValue: minValue(1), maxValue: maxValue(this.balance), integer },
    };
  },
  watch: {
    amountDeposit(DepositCount) {
      if (![1, 5, 10, 50, 100, parseInt(this.balance)].includes(DepositCount)) {
        this.clickedBtn = null;
      } else {
        let index;
        switch (DepositCount) {
          case 1:
            index = parseInt(this.balance) === DepositCount ? 5 : 0;
            break;
          case 5:
            index = parseInt(this.balance) === DepositCount ? 5 : 1;
            break;
          case 10:
            index = parseInt(this.balance) === DepositCount ? 5 : 2;
            break;
          case 50:
            index = parseInt(this.balance) === DepositCount ? 5 : 3;
            break;
          case 100:
            index = parseInt(this.balance) === DepositCount ? 5 : 4;
            break;
          case parseInt(this.balance):
            index = 5;
            break;
        }
        this.clickedBtnChoice(index, DepositCount);
      }
    },
  },
  async created() {
    if (GetCookie("AUTHTOKEN") && GetCookie("SearchToken")) {
      await GetCurrentMoney(GetCookie("AUTHTOKEN"), GetCookie("SearchToken")).then((response) => {
        this.balance = response.currentMoney;
      });
    }
  },
  mounted() {
    this.crashEventHandler = (dataCrash) => {
      try {
        const dataCrashParse = JSON.parse(dataCrash);
        this.crashObject = dataCrashParse;
      } catch (e) {
        console.error(e);
      }
      if (
          this.crashObject.Status === "GameEnd" &&
          this.startGame &&
          this.crashObject.Players.some((player) => player.UserName === GetCookie("SpUserName"))
      ) {
        this.startGame = false;
        setTimeout(() => {
          this.updateUserMoney();
        }, 3000);
      }
      if (
          this.crashObject.Players.some(
              (player) => player.UserName === GetCookie("SpUserName") && player.UserGameState !== "Win"
          ) &&
          !this.startGame &&
          this.crashObject.Status !== "GameEnd"
      ) {
        this.startGame = true;
        const user = this.crashObject.Players.find((player) => player.UserName === GetCookie("SpUserName"));
        if (user) {
          this.amountDeposit = user.Bid;
        }
      }
    };
    eventBus.on("crash", this.crashEventHandler);
  },
  beforeDestroy() {
    eventBus.off("crash", this.crashEventHandler);
  },
  methods: {
    async updateUserMoney() {
      if (GetCookie("AUTHTOKEN") && GetCookie("SearchToken")) {
        await GetCurrentMoney(GetCookie("AUTHTOKEN"), GetCookie("SearchToken")).then((response) => {
          this.balance = response.currentMoney;
          eventBus.emit("Updatebalance");
        });
      }
    },
    async clickPlayBtn() {
      this.v$.$touch();
      this.offBtn = true;
      if (this.v$.amountDeposit.$error) {
        this.textError = "Ошибка введении данных";
        this.offBtn = false;
        this.errorPlayButton();
        return;
      }
      const userData = {
        searchToken: GetCookie("SearchToken"),
        authtoken: GetCookie("AUTHTOKEN"),
      };
      await JoinCrashGame(userData, this.amountDeposit).then((response) => {
        if (
            response === "You can't join to started or ended game" ||
            response === "Player alredy in the game."
        ) {
          this.offBtn = false;
          this.textError =
              response === "You can't join to started or ended game"
                  ? "Игра уже началась или только закончилась!"
                  : "Вы уже в игре!";
          this.errorPlayButton();
          return;
        }
        if (response.ok) {
          this.offBtn = false;
          this.startGame = true;
        }
      });
      this.offBtn = false;
    },
    errorPlayButton() {
      this.ErrorClick = true;
      setTimeout(() => {
        this.ErrorClick = false;
        this.textError = "";
      }, 2000);
    },
    async clickClaimDep() {
      if (!this.startGame) return;
      if (this.isClaiming) return;
      this.isClaiming = true;
      const userData = {
        searchToken: GetCookie("SearchToken"),
        authtoken: GetCookie("AUTHTOKEN"),
      };
      await ExitAndTakeMoneyFromCrashGame(userData)
          .then((response) => {
            console.log(response);
            this.startGame = false;
            setTimeout(() => {
              this.updateUserMoney();
            }, 3000);
          })
          .finally(() => {
            this.isClaiming = false;
          });
    },
    async clickedBtnChoice(index, content) {
      this.clickedBtn = index;
      if (content === "max") {
        await GetCurrentMoney(GetCookie("AUTHTOKEN"), GetCookie("SearchToken")).then((response) => {
          const responseBalance = response.currentMoney;
          if (responseBalance === this.balance) {
            this.amountDeposit = parseInt(responseBalance);
          }
        });
      } else {
        this.amountDeposit = content;
      }
    },
  },
};
</script>

<style scoped>
</style>
