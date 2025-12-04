<template>
  <div class="flex flex-col">
    <LayoutHeader v-if="ticket.doc">
      <template #left-header>
        <Breadcrumbs :items="breadcrumbs" class="breadcrumbs">
          <template #prefix="{ item }">
            <Icon
              v-if="item.icon"
              :icon="item.icon"
              class="mr-1 h-4 flex items-center justify-center self-center"
            />
          </template>
        </Breadcrumbs>
      </template>
      <template #right-header>
        <CustomActions
          v-if="ticket.doc._customActions"
          :actions="ticket.doc._customActions"
        />
        <div v-if="assignees.data?.length">
          <component :is="assignees.data.length == 1 ? 'Button' : 'div'">
            <MultipleAvatar
              :avatars="assignees.data"
              @click="showAssignmentModal = true"
            />
          </component>
        </div>
        <button
          v-else
          class="rounded bg-gray-100 px-2 py-1.5 text-base text-gray-800"
          @click="showAssignmentModal = true"
        >
          Assign
        </button>
        <Dropdown :options="dropdownOptions">
          <template #default="{ open }">
            <Button :label="ticket.doc.status">
              <template #prefix>
                <IndicatorIcon
                  :class="
                    ticketStatusStore.getStatus(ticket.doc.status)?.parsed_color
                  "
                />
              </template>
              <template #suffix>
                <FeatherIcon
                  :name="open ? 'chevron-up' : 'chevron-down'"
                  class="h-4"
                />
              </template>
            </Button>
          </template>
        </Dropdown>
      </template>
    </LayoutHeader>
    <div v-if="ticket.doc" class="flex h-full overflow-hidden">
      <div class="flex flex-1 flex-col max-w-[calc(100%-382px)]">
        <!-- ticket activities -->
        <div class="overflow-y-hidden flex flex-1 !h-full flex-col">
          <Tabs v-model="tabIndex" :tabs="tabs">
            <TabList />
            <TabPanel v-slot="{ tab }" class="h-full">
              <TicketAgentActivities
                ref="ticketAgentActivitiesRef"
                :activities="filterActivities(tab.name)"
                :title="tab.label"
                :ticket-status="ticket.doc?.status"
                @update="
                  () => {
                    ticket.reload();
                  }
                "
                @email:reply="
                  (e) => {
                    communicationAreaRef.replyToEmail(e);
                  }
                "
                @email:forward="
                  (e) => {
                    communicationAreaRef.forwardEmail(e);
                  }
                "
              />
            </TabPanel>
          </Tabs>
        </div>
        <CommunicationArea
          ref="communicationAreaRef"
          :ticketId="String(ticket.doc?.name)"
          :to-emails="[ticket.doc?.raised_by]"
          :cc-emails="[]"
          :bcc-emails="[]"
          :key="ticket.doc?.name"
          @update="
            () => {
              ticket.reload();
              ticketAgentActivitiesRef.scrollToLatestActivity();
            }
          "
        />
      </div>

      <!-- Sidepanel with Resizer -->
      <TicketSidebar />
    </div>
    <AssignmentModal
      v-if="ticket.doc"
      v-model="showAssignmentModal"
      :assignees="assignees.data || []"
      :docname="ticketId"
      :team="ticket.doc?.agent_group"
      doctype="HD Ticket"
      @update="
        () => {
          ticket.reload();
          assignees.reload();
        }
      "
    />
    <SetContactPhoneModal
      v-model="showPhoneModal"
      :name="contact.data?.name"
      @onUpdate="ticket.reload"
    />
  </div>
</template>

<script setup lang="ts">
import {
  AssignmentModal,
  CommunicationArea,
  LayoutHeader,
  MultipleAvatar,
} from "@/components";
import {
  ActivityIcon,
  CommentIcon,
  EmailIcon,
  IndicatorIcon,
  PhoneIcon,
} from "@/components/icons";
import TicketAgentActivities from "@/components/ticket/TicketAgentActivities.vue";
import TicketSidebar from "@/components/ticket-agent/TicketSidebar.vue";
import SetContactPhoneModal from "@/components/ticket/SetContactPhoneModal.vue";
import { useActiveTabManager } from "@/composables/useActiveTabManager";
import { useActiveViewers } from "@/composables/realtime";
import { useTicket } from "@/composables/useTicket";
import { ticketsToNavigate } from "@/composables/useTicketNavigation";
import { useView } from "@/composables/useView";
import { globalStore } from "@/stores/globalStore";
import { useTelephonyStore } from "@/stores/telephony";
import { useTicketStatusStore } from "@/stores/ticketStatus";
import {
  ActivitiesSymbol,
  AssigneeSymbol,
  Customizations,
  CustomizationSymbol,
  RecentSimilarTicketsSymbol,
  Resource,
  TabObject,
  TicketContactSymbol,
  TicketTab,
  TicketSymbol,
  View,
} from "@/types";
import { HDTicketStatus } from "@/types/doctypes";
import { getIcon } from "@/utils";
import {
  Breadcrumbs,
  Dropdown,
  TabList,
  TabPanel,
  Tabs,
  createResource,
  toast,
  usePageMeta,
} from "frappe-ui";
import { storeToRefs } from "pinia";
import {
  ComputedRef,
  computed,
  h,
  onBeforeUnmount,
  onMounted,
  provide,
  ref,
  watch,
} from "vue";
import { useRoute } from "vue-router";
import { showCommentBox, showEmailBox } from "./modalStates";

const telephonyStore = useTelephonyStore();
const { isCallingEnabled } = storeToRefs(telephonyStore);
const ticketStatusStore = useTicketStatusStore();
const { $socket } = globalStore();
const { findView } = useView("HD Ticket");

const props = defineProps({
  ticketId: {
    type: String,
    required: true,
  },
});
const route = useRoute();
const showPhoneModal = ref(false);
const showAssignmentModal = ref(false);
const ticketAgentActivitiesRef = ref(null);
const communicationAreaRef = ref(null);

const ticketComposable = computed(() => useTicket(props.ticketId));
const ticket = computed(() => ticketComposable.value.ticket);
const assignees = computed(() => ticketComposable.value.assignees);
const contact = computed(() => ticketComposable.value.contact);
const activities = computed(() => ticketComposable.value.activities);
const customizations: Resource<Customizations> = createResource({
  url: "helpdesk.helpdesk.doctype.hd_ticket.api.get_ticket_customizations",
  cache: ["HD Ticket", "customizations"],
  auto: true,
});

provide(TicketSymbol, ticket);
provide(
  AssigneeSymbol,
  computed(() => ticketComposable.value.assignees)
);
provide(
  TicketContactSymbol,
  computed(() => ticketComposable.value.contact)
);
provide(
  CustomizationSymbol,
  computed(() => customizations)
);
provide(
  RecentSimilarTicketsSymbol,
  computed(() => ticketComposable.value.recentSimilarTickets)
);
provide(
  ActivitiesSymbol,
  computed(() => ticketComposable.value.activities)
);
provide("communicationArea", communicationAreaRef);
provide("makeCall", () => {
  if (
    !contact.value.data?.mobile_no &&
    !contact.value.data?.phone
  ) {
    showPhoneModal.value = true;
    return;
  }
  telephonyStore.makeCall({
    number:
      contact.value.data?.phone ||
      contact.value.data?.mobile_no,
    doctype: "HD Ticket",
    docname: props.ticketId,
  });
});
const viewerComposable = computed(() => useActiveViewers(ticket.value?.doc?.name || props.ticketId));
const viewers = computed(
  () => viewerComposable.value.currentViewers[props.ticketId] || []
);
const { startViewing, stopViewing } = viewerComposable.value;

const breadcrumbs = computed(() => {
  let items = [{ label: "Tickets", route: { name: "TicketsAgent" } }];
  if (route.query.view) {
    const currView: ComputedRef<View> = findView(route.query.view as string);
    if (currView.value) {
      items.push({
        label: currView.value?.label,
        icon: getIcon(currView.value?.icon),
        route: { name: "TicketsAgent", query: { view: currView.value?.name } },
      });
    }
  }
  items.push({
    label: ticket.value.doc?.subject || props.ticketId,
  });
  return items;
});

const dropdownOptions = computed(() =>
  ticketStatusStore.statuses.data
    ?.filter((s: HDTicketStatus) => s.enabled)
    .map((o: HDTicketStatus) => ({
      label: o.label_agent,
      value: o.label_agent,
      onClick: () => {
        if (ticket.value.doc.status === o.label_agent) return;
        ticket.value.setValue.submit(
          { status: o.label_agent },
          {
            onSuccess() {
              activities.value.reload();
            },
          }
        );
      },
      icon: () =>
        h(IndicatorIcon, {
          class: o.parsed_color,
        }),
    })) || []
);

const tabs: ComputedRef<TabObject[]> = computed(() => {
  const _tabs: TabObject[] = [
    {
      name: "activity",
      label: "Activity",
      icon: ActivityIcon,
    },
    {
      name: "email",
      label: "Emails",
      icon: EmailIcon,
    },
    {
      name: "comment",
      label: "Comments",
      icon: CommentIcon,
    },
  ];

  if (isCallingEnabled.value) {
    _tabs.push({
      name: "call",
      label: "Calls",
      icon: PhoneIcon,
    });
  }
  return _tabs;
});

const { tabIndex, changeTabTo } = useActiveTabManager(tabs);

// Process activities similar to TicketActivityPanel
const _activities = computed(() => {
  if (!activities.value?.data) {
    return [];
  }

  const emailProps = activities.value?.data?.communications.map(
    (email, idx: number) => {
      return {
        subject: email.subject,
        content: email.content,
        sender: { name: email.user.email, full_name: email.user.name },
        to: email.recipients,
        type: "email",
        key: email.creation,
        cc: email.cc,
        bcc: email.bcc,
        creation: email.communication_date || email.creation,
        attachments: email.attachments,
        name: email.name,
        deliveryStatus: email.delivery_status,
        isFirstEmail: idx === 0,
      };
    }
  ) || [];

  const commentProps = activities.value.data.comments.map((comment) => {
    return {
      name: comment.name,
      type: "comment",
      key: comment.creation,
      commentedBy: comment.commented_by,
      commenter: comment.user.name,
      creation: comment.creation,
      content: comment.content,
      attachments: comment.attachments,
    };
  }) || [];

  const historyProps = [
    ...(activities.value.data.history || []),
    ...(activities.value.data.views || []),
  ].map((h) => {
    return {
      type: "history",
      key: h.creation,
      content: h.action ? h.action : "viewed this",
      creation: h.creation,
      user: h.user.name + " ",
    };
  });

  const callProps = (activities.value.data.calls || []).map((call) => {
    return {
      ...call,
      type: "call",
      name: call.name,
      key: call.creation,
      call_type: call.type,
      content: `${call.caller || "Unknown"} made a call to ${
        call.receiver || "Unknown"
      }`,
      duration: call.duration ? call.duration + "s" : "0s",
    };
  });

  const sorted = [
    ...emailProps,
    ...commentProps,
    ...historyProps,
    ...callProps,
  ].sort((a, b) => new Date(a.creation) - new Date(b.creation));

  const data = [];
  let i = 0;

  while (i < sorted.length) {
    const currentActivity = sorted[i];

    if (currentActivity.type === "history") {
      currentActivity.relatedActivities = [currentActivity];
      for (let j = i + 1; j < sorted.length + 1; j++) {
        const nextActivity = sorted[j];

        if (
          nextActivity &&
          nextActivity.user === currentActivity.user &&
          nextActivity.content !== "viewed this" &&
          !nextActivity.content.includes("assigned") &&
          !nextActivity.content.includes("unassigned")
        ) {
          currentActivity.relatedActivities.push(nextActivity);
        } else {
          data.push(currentActivity);
          i = j - 1;
          break;
        }
      }
    } else {
      data.push(currentActivity);
    }
    i++;
  }

  // Add feedback if exists
  if (ticket.value?.doc?.feedback_rating === 0 || !ticket.value?.doc?.feedback_rating) {
    return data;
  }
  let feedbackActivity = [
    {
      type: "feedback",
      key: "feedback-activity",
      feedback_rating: ticket.value?.doc.feedback_rating,
      feedback_extra: ticket.value?.doc.feedback_extra,
      feedback: ticket.value?.doc.feedback,
      sender: {
        name: ticket.value?.doc.raised_by,
        full_name: ticket.value?.doc.contact,
      },
    },
  ];
  data.push(...feedbackActivity);

  return data;
});

function filterActivities(eventType: TicketTab) {
  if (eventType === "activity") {
    return _activities.value;
  }
  return _activities.value.filter((activity) => activity.type === eventType);
}

// handling for faster navigation between tickets
watch(
  () => route.params.ticketId,
  (newTicketId, oldTicketId) => {
    if (newTicketId === oldTicketId) return;

    if (oldTicketId) stopViewing(oldTicketId as string);
    startViewing(newTicketId as string);
  },
  { immediate: true }
);

type TicketUpdateData = {
  ticket_id: string;
  user: string;
  field: string;
  value: string;
};

onMounted(() => {
  ticketsToNavigate.update({
    params: {
      ticket: props.ticketId,
      current_view: route.query.view as string,
    },
  });
  ticketsToNavigate.reload();
  if (ticket.value?.markSeen) {
    ticket.value.markSeen.reload();
  }
  startViewing(props.ticketId);

  $socket.on("ticket_update", (data: TicketUpdateData) => {
    if (data.ticket_id === ticket.value?.doc?.name || data.ticket_id === props.ticketId) {
      // Notify the user about the update
      toast.info(`User ${data.user} updated ${data.field} to ${data.value}`);
    }
  });
});

onBeforeUnmount(() => {
  stopViewing(props.ticketId);
  showEmailBox.value = false;
  showCommentBox.value = false;
});

usePageMeta(() => {
  return {
    title: props.ticketId,
  };
});
</script>

<style>
.breadcrumbs button {
  background-color: inherit !important;
  &:hover,
  &:focus {
    background-color: inherit !important;
  }
}
</style>
