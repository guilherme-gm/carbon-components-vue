<template>
  <cv-wrapper
    :id="cvId"
    :tag-type="formItem ? 'div' : ''"
    :class="`cv-date-picker ${carbonPrefix}--form-item`"
  >
    <div
      :id="formItem ? undefined : cvId"
      ref="dateWrapper"
      :data-date-picker="['single', 'range'].includes(getKind)"
      :data-date-picker-type="getKind"
      :class="[
        `${carbonPrefix}--date-picker ${carbonPrefix}--date-picker--${getKind}`,
        {
          [`${carbonPrefix}--date-picker--simple`]: getKind === 'short',
          [`${carbonPrefix}--date-picker--light`]: isLight,
          'cv-date-picker': !formItem,
        },
      ]"
    >
      <div
        :class="{
          [`${carbonPrefix}--date-picker-container`]: [
            'single',
            'range',
          ].includes(getKind),
          [`${carbonPrefix}--date-picker--nolabel`]: !getDateLabel,
        }"
      >
        <label
          v-if="getDateLabel.length > 0"
          :for="`${cvId}-input-1`"
          :class="`${carbonPrefix}--label`"
        >
          {{ getDateLabel }}
        </label>
        <div :class="`${carbonPrefix}--date-picker-input__wrapper`">
          <input
            :id="`${cvId}-input-1`"
            ref="date"
            type="text"
            data-date-picker-input
            data-testid="datepicker"
            :aria-description="placeholder"
            :data-invalid="isInvalid || null"
            :disabled="disabled || null"
            :data-date-picker-input-from="`${getKind === 'range'}`"
            :class="`${carbonPrefix}--date-picker__input`"
            :pattern="pattern"
            :placeholder="placeholder"
            @change="handleUpdateEvent"
          />
          <Calendar16
            v-if="['single', 'range'].includes(getKind)"
            :class="`${carbonPrefix}--date-picker__icon`"
            data-date-picker-icon
          />
        </div>

        <div v-if="isInvalid" :class="`${carbonPrefix}--form-requirement`">
          <slot name="invalid-message">{{ invalidMessage }}</slot>
        </div>
      </div>

      <!-- maybe use input as another component? -->
      <div
        v-if="isRange"
        :class="{
          [`${carbonPrefix}--date-picker-container`]: getKind === 'range',
          [`${carbonPrefix}--date-picker--nolabel`]: dateEndLabel !== undefined,
        }"
      >
        <label
          v-if="dateEndLabel.length > 0"
          :for="`${cvId}-input-2`"
          :class="`${carbonPrefix}--label`"
        >
          {{ dateEndLabel }}
        </label>
        <div
          :class="`${carbonPrefix}--date-picker-input__wrapper`"
          @click="calendar.open()"
        >
          <input
            :id="`${cvId}-input-2`"
            ref="todate"
            type="text"
            data-date-picker-input
            data-testid="todatepicker"
            :aria-description="placeholder"
            :data-date-picker-input-to="`${kind === 'range'}`"
            :data-invalid="isInvalid || null"
            :disabled="disabled || null"
            :class="`${carbonPrefix}--date-picker__input`"
            :pattern="pattern"
            :placeholder="placeholder"
            @change="handleUpdateEvent"
          />
          <Calendar16
            :class="`${carbonPrefix}--date-picker__icon`"
            data-date-picker-icon
          />
        </div>
        <div v-if="isInvalid" :class="`${carbonPrefix}--form-requirement`">
          <span id="invalid-message-placeholder"></span>
        </div>
      </div>
    </div>
  </cv-wrapper>
</template>

<script setup>
import {
  ref,
  computed,
  onMounted,
  onBeforeMount,
  onUnmounted,
  watch,
  nextTick,
  useSlots,
} from 'vue';
import { carbonPrefix } from '../../global/settings';
import { props as propsCvId, useCvId } from '../../use/cvId';
import { Calendar16 } from '@carbon/icons-vue';
import CvWrapper from '../CvWrapper/CvWrapper';
import { props as propsCvTheme, useIsLight } from '../../use/cvTheme';
import flatpickr from 'flatpickr';
import l10n from 'flatpickr/dist/l10n/index';
import carbonFlatpickrFixEventsPlugin from './plugins/fixEventsPlugin';
import carbonFlatpickrRangePlugin from './plugins/rangePlugin';
import carbonFlatpickrMonthSelectPlugin from './plugins/monthSelectPlugin';

const slots = useSlots();

const dateWrapper = ref(null);
const date = ref(null);
const todate = ref(null);

const calendar = ref(null);

const props = defineProps({
  modelValue: { type: [String, Object, Array, Date], default: undefined },
  /** Date picker label */
  dateLabel: { type: String, default: undefined },
  /** Date picker end label (when using kind="range") */
  dateEndLabel: { type: String, default: 'End date' },
  /** If true, the date picker will be disabled */
  disabled: { type: Boolean, default: false },
  /** Date picker text on invalid value */
  invalidMessage: { type: String, default: undefined },
  /** Regex pattern used for form validation, default \d{1,2}/\d{1,2}/\d{4} */
  pattern: { type: String, default: '\\d{1,2}/\\d{1,2}/\\d{4}' },
  /** Date picker input placeholder, shown when date picker is empty. Default 'mm/dd/yyyy' */
  placeholder: { type: String, default: 'mm/dd/yyyy' },
  /** You can pass flatpickr options through this prop.
   See https://flatpickr.js.org/options/ for more details.
   Some of the options is not supported (for example, onChange, onReady, mode, nextArrow, prevArrow).
   Also, you have to pass dateFormat field in calOptions object if you use it. */
  calOptions: {
    type: Object,
    default() {
      return {
        dateFormat: 'm/d/Y',
      };
    },
  },
  formItem: { type: Boolean, default: true },
  /** Date picker kind (also known as mode in flatpickr options).
   Possible values: 'short', 'simple', 'single', 'range' */
  kind: {
    type: String,
    default: 'simple',
    validator: val => ['short', 'simple', 'single', 'range'].includes(val),
  },
  value: { type: [String, Object, Array, Date], default: undefined },
  ...propsCvId,
  ...propsCvTheme,
});

const cvId = useCvId(props, true, 'date-picker-');
const isLight = useIsLight(props);

const emit = defineEmits(['update:modelValue', 'dateChange']);

const getKind = computed({
  get() {
    if (['short', 'simple', 'single', 'range'].includes(props.kind)) {
      return props.kind;
    }

    return 'simple';
  },
});

const getDateLabel = computed(() => {
  if (props.dateLabel !== undefined) {
    return props.dateLabel;
  } else {
    if (props.kind === 'range') {
      return 'Start date';
    } else {
      return 'Date';
    }
  }
});

const isRange = computed(() => {
  return props.kind === 'range';
});

const isSingle = computed(() => {
  return props.kind === 'single';
});

const isInvalid = computed(() => {
  return !!props.invalidMessage || !!slots['invalid-message'];
});

const getFlatpickrOptions = () => {
  const options = { ...props.calOptions };

  options.plugins = [
    props.kind === 'range'
      ? carbonFlatpickrRangePlugin({
          input: todate.value,
        })
      : () => {},
    carbonFlatpickrMonthSelectPlugin({
      selectorFlatpickrMonthYearContainer: '.flatpickr-current-month',
      selectorFlatpickrYearContainer: '.numInputWrapper',
      selectorFlatpickrCurrentMonth: '.cur-month',
      classFlatpickrCurrentMonth: 'cur-month',
    }),
    carbonFlatpickrFixEventsPlugin({
      inputFrom: date.value,
      inputTo: todate.value,
    }),
  ];

  options.nextArrow = `
    <svg width="16px" height="16px" viewBox="0 0 16 16">
      <polygon points="11,8 6,13 5.3,12.3 9.6,8 5.3,3.7 6,3 "/>
      <rect width="16" height="16" style="fill:none" />
    </svg>
  `;

  options.prevArrow = `
    <svg width="16px" height="16px" viewBox="0 0 16 16">
      <polygon points="5,8 10,3 10.7,3.7 6.4,8 10.7,12.3 10,13 "/>
      <rect width="16" height="16" style="fill:none" />
    </svg>
  `;

  options.mode = props.kind;
  // add events update based on parameters
  options.onChange = handleDatePick;
  // options.onOpen = onOpen;
  options.onReady = onCalReady;

  return options;
};

const onCalReady = (selectedDates, dateStr, instance) => {
  const calendarContainer = instance.calendarContainer;
  const options = {
    classCalendarContainer: `${carbonPrefix}--date-picker__calendar`,
    classMonth: `${carbonPrefix}--date-picker__month`,
    classWeekdays: `${carbonPrefix}--date-picker__weekdays`,
    classDays: `${carbonPrefix}--date-picker__days`,
    classWeekday: `${carbonPrefix}--date-picker__weekday`,
    classDay: `${carbonPrefix}--date-picker__day`,
    classFocused: `${carbonPrefix}--focused`,
    classVisuallyHidden: `${carbonPrefix}--visually-hidden`,
  };

  if (calendarContainer) {
    calendarContainer.classList.add(options.classCalendarContainer);
    calendarContainer
      .querySelector('.flatpickr-month')
      .classList.add(options.classMonth);
    calendarContainer
      .querySelector('.flatpickr-weekdays')
      .classList.add(options.classWeekdays);
    calendarContainer
      .querySelector('.flatpickr-days')
      .classList.add(options.classDays);
    for (const item of calendarContainer.querySelectorAll(
      '.flatpickr-weekday'
    )) {
      const currentItem = item;
      currentItem.innerHTML = currentItem.innerHTML.replace(/\s+/g, '');
      currentItem.classList.add(options.classWeekday);
    }
    for (const item of calendarContainer.querySelectorAll('.flatpickr-day')) {
      item.classList.add(options.classDay);
      if (item.classList.contains('today') && selectedDates.length > 0) {
        item.classList.add('no-border');
      } else if (
        item.classList.contains('today') &&
        selectedDates.length === 0
      ) {
        item.classList.remove('no-border');
      }
    }
  }
};

const initFlatpickr = () => {
  return flatpickr(date.value, getFlatpickrOptions());
};
watch(
  () => props.calOptions,
  () => {
    for (const [key, value] of Object.entries(props.calOptions)) {
      calendar.value?.set(key, value);
    }
  },
  { deep: true }
);

let dateToString = val => {
  if (!val) return '';
  return calendar.value?.formatDate(val, props.calOptions.dateFormat);
};

// eslint-disable-next-line no-unused-vars
const handleDatePick = (selectedDates, dateStr, instance) => {
  if (isSingle.value) {
    const temp = dateToString(selectedDates[0]);

    nextTick(() => {
      date.value.value = temp;
    });

    emit('update:modelValue', temp);
    emit('dateChange', selectedDates);
  } else if (isRange.value && selectedDates[0] && selectedDates[1]) {
    const startDate = dateToString(selectedDates[0]);
    const endDate = dateToString(selectedDates[1]);

    nextTick(() => {
      date.value.value = startDate;
      todate.value.value = endDate;
    });

    emit('update:modelValue', {
      startDate: startDate,
      endDate: endDate,
    });
    emit('dateChange', selectedDates);
  }
};

const handleUpdateEvent = event => {
  if (['short', 'simple'].includes(props.kind)) {
    emit('update:modelValue', event.target.value);
  }
};

const setDate = value => {
  try {
    if (value === undefined) return;
    if (isSingle.value) {
      calendar.value?.setDate(value, true);
    } else if (isRange.value) {
      calendar.value?.setDate([value.startDate, value.endDate], true);
    } else {
      date.value.value = value;
    }
  } catch (e) {
    console.error('setDate', e);
  }
};

watch(
  () => props.modelValue,
  newValue => {
    if (isRange.value) {
      if (
        date.value.value !== newValue.startDate ||
        todate.value.value !== newValue.endDate
      ) {
        setDate(newValue);
      }
    } else {
      if (date.value.value !== newValue) {
        setDate(newValue);
      }
    }
  },
  { deep: true }
);

onBeforeMount(() => {
  // Weekdays shorthand for english locale
  l10n.en.weekdays.shorthand.forEach((day, index) => {
    const currentDay = l10n.en.weekdays.shorthand;
    if (currentDay[index] === 'Thu' || currentDay[index] === 'Th') {
      currentDay[index] = 'Th';
    } else {
      currentDay[index] = currentDay[index].charAt(0);
    }
  });
});

onMounted(() => {
  if (['range', 'single'].includes(props.kind)) {
    calendar.value = initFlatpickr();
  }

  setDate(props.modelValue || props.value);
});

onUnmounted(() => {
  calendar.value?.destroy();
});
</script>

<style lang="scss">
#invalid-message-placeholder {
  display: block;
  height: 16px;
}
</style>
